# CCI Lite — MVP Architecture Documentation

This README defines the **canonical, locked architecture** for the Conversant CCI Lite MVP. It includes **Phase 0**, **Phase 1**, and the newly completed **Phase 2**. All sections are written to match the exact, working AWS + Clerk implementation.

---

# 🚀 Overview

CCI Lite MVP is a lightweight, multi-tenant, post-call analytics engine powered by:

* **Clerk** → Authentication + tenant isolation
* **AWS** → Ingestion, transcription, enrichment, AI analysis
* **S3** → Canonical storage per tenant
* **Replit Backend** → Reads tenant S3 data
* **Replit Frontend** → Multi-tenant dashboard

The system is deliberately simple:

* **No databases**
* **No backend provisioning logic**
* **No complex user management**
* **Each tenant = one Clerk organization**
* **All data isolation = S3 prefixes**
* **All provider credentials = Secrets Manager**

This architecture is now **frozen** for MVP.

---

# ⭐ PHASE 0 — Foundation Lock-In (Complete & Locked)

Phase 0 locks in the entire architectural foundation, ensuring future phases cannot drift.

---

## **0.1 Tenant Identity = Clerk Organization**

Every customer is represented by one Clerk **Organization**.
Session tokens include the required fields:

```json
{
  "org_id": "{{organization.id}}",
  "org_name": "{{organization.name}}",
  "user_id": "{{user.id}}",
  "email": "{{user.primary_email_address}}"
}
```

These values power:

* Tenant isolation
* S3 directory mapping
* Dashboard routing
* Multi-user access control

---

## **0.2 S3 = The Tenant Database**

Each tenant receives a dedicated directory in every S3 bucket.

### Input Bucket

```
<org_id>/audio/
<org_id>/meta/
```

### Results Bucket

```
<org_id>/raw/
<org_id>/enriched/
<org_id>/final/
    v1.0/
    v2.0-flat/calls/
    v2.0-flat/qa/
```

### Config Bucket

```
<org_id>/ai-config.json
<org_id>/qa-config.json
```

**There is no database — S3 *is* the datastore.**

---

## **0.3 Tenant Config = S3 JSON Files**

Per-tenant AI instructions and QA rubrics live inside:

```
cci-lite-config/<org_id>/
    ai-config.json
    qa-config.json
```

These files can fully personalise summary behaviour and QA criteria.

---

## **0.4 Provider Credentials = Secrets Manager**

Every customer’s call recording provider keys are stored under:

```
cci-lite/<provider>/<tenant_id>
```

Example for ClarifyGo/Oak:

```
cci-lite/oak/demo-tenant
```

Secrets contain credentials such as:

```json
{
  "base_url": "...",
  "client_id": "...",
  "client_secret": "...",
  "username": "...",
  "password": "...",
  "token_path": "...",
  "recordings_path": "...",
  "recording_download_path": "..."
}
```

These are created manually during onboarding.

---

## **0.5 AWS Data Pipeline (Locked)**

The ingestion + analytics pipeline is:

```
Recording Provider (ClarifyGo/Oak)
   → cci-lite-oak-ingest (Lambda — multi-tenant)
        → S3 input/<org_id>/audio + meta
            → cci-lite-job-init
                → Transcribe
                → S3 results/<org_id>/raw
                    → cci-lite-result-handler
                        → Comprehend
                        → S3 results/<org_id>/enriched
                            → cci-lite-analyser
                                → Bedrock + metadata join
                                → S3 results/<org_id>/final/
                                    v1.0/
                                    v2.0-flat/calls/
                                    v2.0-flat/qa/
```

This pipeline has been tested end-to-end and is stable.

---

# ⭐ PHASE 1 — AWS Infrastructure Confirmation (Complete & Locked)

Phase 1 validated that all AWS components work together with Clerk-only tenancy.

---

## **1.1 S3 Buckets (Confirmed)**

Region: `eu-central-1`

```
cci-lite-input-<account>-eu-central-1
cci-lite-results-<account>-eu-central-1
cci-lite-config-<account>-eu-central-1
```

All three buckets contain prefixes for `<org_id>/`.

---

## **1.2 Lambda Functions (Confirmed)**

Required functions:

```
cci-lite-oak-ingest
cci-lite-job-init
cci-lite-result-handler
cci-lite-analyser
```

All functions:

* Have correct environment variables
* Have correct triggers (S3, SQS, EventBridge)
* Produce valid logs
* Operate end-to-end

---

## **1.3 EventBridge (Confirmed)**

ClarifyGo/Oak ingestion uses a 15-minute schedule:

```
rate(15 minutes)
```

---

## **1.4 IAM Roles (Confirmed)**

`cci-lite-lambda-role` includes:

* S3 read/write
* Secrets Manager List/Get
* KMS decrypt/encrypt
* Lambda logs
* Transcribe
* Comprehend
* Bedrock
* SQS

All permissions validated with live pipeline execution.

---

## **1.5 Secrets Manager (Confirmed)**

Working tenant secret example:

```
cci-lite/oak/demo-tenant
```

Contains required ClarifyGo/Oak credentials.

---

## **1.6 CloudWatch Logs (Confirmed)**

Log groups exist for all Lambdas and show correct event flow.

---

## **Phase 1 Status: COMPLETE & LOCKED**

---

# ⭐ PHASE 2 — Multi-Tenant Provider Ingestion (Newly Completed)

Phase 2 replaces the original single-tenant ingestion model with a clean, scalable, **provider-specific multi-tenant ingestion system**.

This is now the official ingestion architecture for CCI Lite.

---

## **2.1 Design Principles**

1. **One ingestion Lambda per provider**

   * `cci-lite-oak-ingest` for ClarifyGo/Oak
   * Future: `cci-lite-3cx-ingest`, `cci-lite-asc-ingest`, etc.

2. **Each provider Lambda is multi-tenant**

   * Discovers all tenants for that provider
   * Loads each tenant's credentials from Secrets Manager
   * Runs ingestion independently per tenant

3. **Secrets Manager is the single source of truth**

   * No databases
   * No Supabase
   * No backend config store

4. **S3 is the processing pipeline trigger**

   * Audio → triggers Transcribe
   * Raw → triggers Comprehend
   * Enriched → triggers Analyzer

5. **Clerk org_id = tenant_id = S3 prefix**

   * All data fully isolated per tenant

---

## **2.2 Multi-Tenant Discovery Logic (Final)**

Tenants are fetched by listing secrets and filtering by prefix:

```python
def discover_tenants():
    prefix = "cci-lite/oak/"
    resp = secrets.list_secrets()

    tenants = []
    for sec in resp.get("SecretList", []):
        name = sec["Name"]
        if name.startswith(prefix):
            tenant_id = name.replace(prefix, "")
            tenants.append(tenant_id)

    return tenants
```

This approach is stable and AWS-correct.

---

## **2.3 Per-Tenant Credential Loading**

```python
def load_credentials(tenant_id):
    secret_name = f"cci-lite/oak/{tenant_id}"
    raw = secrets.get_secret_value(SecretId=secret_name)
    return json.loads(raw["SecretString"])
```

Manual onboarding guarantees secret names match exactly.

---

## **2.4 Updated IAM Policy (Final)**

Critical change:

```
secretsmanager:ListSecrets → Resource: "*"
secretsmanager:GetSecretValue → Resource: "arn:aws:secretsmanager:...:secret:cci-lite/*"
```

This is required because AWS does not support prefix restrictions on ListSecrets.

---

## **2.5 Updated Provider Ingestion Lambda (Summary)**

Changes implemented:

* Converted ingestion to multi-tenant loop
* Removed TENANT_ID from env vars
* Removed OAK_SECRET_ARN from env vars
* Added dynamic tenant discovery
* Added dynamic credential loading
* Preserved 100% of ClarifyGo/Oak logic
* Ensured S3 writes use `<tenant_id>/...` prefix

The Lambda is now:

* **Provider-specific**
* **Multi-tenant**
* **Fully isolated per customer**
* **Ready for additional providers**

---

## **2.6 Onboarding Flow (Manual MVP Process)**

For each new customer:

1. Create Clerk Organization
2. Create S3 prefixes:

```
<tenant>/audio
<tenant>/meta
<tenant>/raw
<tenant>/enriched
<tenant>/final/v1.0
<tenant>/final/v2.0-flat/calls
<tenant>/final/v2.0-flat/qa
```

3. Create config files in config bucket:

```
<tenant>/ai-config.json
<tenant>/qa-config.json
```

4. Create provider credentials:

```
cci-lite/oak/<tenant>
```

5. EventBridge triggers ingestion automatically.
6. Dashboard reads results from S3.

This delivers **automated ingestion with zero provisioning automation**.

---

## **Phase 2 Status: COMPLETE & LOCKED**

Multi-tenant ingestion is now fully operational and validated.

---

# Next Steps

* **Phase 3 — Backend API (S3 → Frontend Gateway)**
* **Phase 4 — Frontend Multi-Tenant Dashboard**
* **Phase 5 — Tenant Onboarding Manual**
* **Phase 6 — QA & Internal Testing**
* **Phase 7 — Pilot Deployment**
* **Phase 8 — Public Launch**

This README will continue to be updated phase by phase.
