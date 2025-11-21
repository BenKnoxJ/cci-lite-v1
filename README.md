# CCI Lite MVP Build — Phase 0 & Phase 1 Documentation (Clerk‑Only Architecture)

This document defines the **authoritative, locked foundation** for the CCI Lite MVP.
It replaces the previous Supabase-dependent design with a **pure Clerk + AWS** multi‑tenant architecture.

This document is intended to live as the root **README.md** for the GitHub repository and will grow as each new phase is completed.

---

# 🚀 Overview

CCI Lite MVP is a lightweight, multi‑tenant, post‑call analytics platform powered by:

* **Clerk** → Authentication + tenant (organization) ownership + user roles
* **AWS** → Full data processing pipeline (S3, Lambda, EventBridge, Secrets Manager, Transcribe, Comprehend, Bedrock)
* **S3** → The single source-of-truth for tenant call data and configuration
* **Replit Backend** → Reads enriched outputs from S3 and serves them to the frontend
* **Replit Frontend** → Multi‑tenant dashboard, isolated per Clerk organization

**No databases are used in MVP**.
All tenancy is derived from Clerk organizations.
All call data is stored and processed in AWS.

This eliminates unnecessary complexity and accelerates delivery.

---

# ⭐ PHASE 0 — Foundation Lock-In (Clerk-Only Architecture)

Phase 0 ensures the entire high-level architecture is locked and cannot drift.
This guarantees a stable foundation for all future phases.

## ✅ Deliverables

* Authentication model locked (Clerk only)
* Multi‑tenant model locked (Clerk organizations)
* Data flow locked (AWS → S3 → Dashboard)
* No database dependency
* Dashboard requirements frozen

---

## ✅ 1. Clerk = Authentication + Tenant Ownership

Clerk Organizations act as **tenants**.
Each user belongs to exactly one Clerk Organization, which determines their data access.

### Required session token claims

Configured in Clerk → Customize Session Token:

```json
{
  "org_id": "{{organization.id}}",
  "org_name": "{{organization.name}}",
  "user_id": "{{user.id}}",
  "email": "{{user.primary_email_address}}"
}
```

These four values give the backend everything it needs:

* `org_id` → used as S3 prefix (tenant isolation)
* `org_name` → used for UI
* `user_id` → identifies the logged-in user
* `email` → for admin-level visibility later

No webhooks.
No provisioning automation.
No syncing with a database.

---

## ✅ 2. Tenant Isolation = S3 Prefix Per Clerk Organization

Each tenant (Clerk organization) has a dedicated directory in all S3 buckets.

Example for tenant `demo-tenant`:

### Input Bucket (`cci-lite-input-*`)

```
demo-tenant/
    audio/
    meta/
```

### Results Bucket (`cci-lite-results-*`)

```
demo-tenant/
    raw/
    enriched/
    final/
        v1.0/
        v2.0-flat/
            calls/
            qa/
```

### Config Bucket (`cci-lite-config-*`)

```
demo-tenant/
    ai-config.json
    qa-config.json
```

The frontend backend simply uses:

```
s3://<bucket>/<org_id>/...
```

No database lookups required.

---

## ✅ 3. Tenant Settings = Stored in S3

All tenant customisation is stored in S3:

* **ai-config.json** → AI instructions for summary + next steps
* **qa-config.json** → QA rubric & criteria

These can be customised per tenant by editing a JSON file — no database needed.

---

## ✅ 4. Secrets Manager = Provider Credentials Per Tenant

Each customer’s recording provider keys are stored in Secrets Manager.
Example:

```
cci-lite/oak/<tenant>
```

This contains the API credentials needed for ingestion.

The ingestion Lambda loads:

```
TENANT_ID (from env var)
OAK_SECRET_ARN (per tenant)
```

For additional tenants:
➜ A new secret is created manually.

---

## ✅ 5. Data Pipeline Architecture (Locked)

The AWS pipeline processes calls as follows:

```
Recording Provider (OAK / ClarifyGo)
   → cci-lite-oak-ingest (Lambda)
       → S3 input/<org_id>/audio + meta
           → cci-lite-job-init (Lambda)
               → Transcribe
               → S3 results/<org_id>/raw
                   → cci-lite-result-handler (Lambda)
                       → Comprehend
                       → S3 results/<org_id>/enriched
                           → cci-lite-analyser (Lambda)
                               → Bedrock + join metadata
                               → S3 results/<org_id>/final/
                                   v1.0/
                                   v2.0-flat/calls/
                                   v2.0-flat/qa/
```

This flow is locked and fully operational.

---

## ✅ 6. Dashboard Requirements (Frozen for MVP)

Dashboard reads **only** enriched and flattened files from S3.

Sections:

* **Business Overview**
* **QA Overview**
* **Agent Performance**
* **Calls Explorer**
* **Call Detail View**

All metrics are derived from:

* `final/v1.0/*.json`
* `final/v2.0-flat/calls/*.jsonl`
* `final/v2.0-flat/qa/*.jsonl`

No database queries.
No backend joins.

**Phase 0 Status: COMPLETE & LOCKED**

---

# ⭐ PHASE 1 — AWS Infrastructure Confirmation

Phase 1 validates that AWS is aligned with the Clerk-only tenant model and fully functional.

## 1. S3 Buckets (Confirmed)

Region: **eu-central-1 (Frankfurt)**

```
cci-lite-config-591338347562-eu-central-1
cci-lite-input-591338347562-eu-central-1
cci-lite-results-591338347562-eu-central-1
```

Tenant prefix used: `demo-tenant/`

---

## 2. Lambda Functions (Confirmed)

All required pipeline functions are deployed:

```
cci-lite-oak-ingest
cci-lite-job-init
cci-lite-result-handler
cci-lite-analyser
```

All runtime versions, environment variables, outputs, and triggers are correct and validated.

### Pipeline Summary

#### cci-lite-oak-ingest

* Pulls call recordings
* Writes to `input/<org_id>/audio` and `meta`
* Triggered by EventBridge schedule

#### cci-lite-job-init

* Triggered on new audio files
* Transcribes audio → raw transcript JSON
* Writes to `results/<org_id>/raw`

#### cci-lite-result-handler

* Triggered on raw transcripts
* Runs Comprehend enrichment
* Writes to `results/<org_id>/enriched`

#### cci-lite-analyser

* Triggered by SQS
* Runs Bedrock + metadata join
* Writes final outputs to `results/<org_id>/final/`

All Lambdas are working end-to-end.

---

## 3. EventBridge (Confirmed)

A scheduled rule triggers ingestion:

```
cci-lite-oak-ingest-schedule
rate(15 minutes)
```

Fully enabled.

---

## 4. IAM Roles (Confirmed)

Policies attached include:

* AmazonTranscribeFullAccess
* AWSLambdaBasicExecutionRole
* cci-lite-lambda-policy
* cci-lite-analyzer-policy
* CCI-Lite-OAKIngestPermissions
* sqs-trigger-policy

Supports:

* S3 read/write
* Secrets Manager
* Transcribe
* Comprehend
* Bedrock
* SQS

---

## 5. Secrets Manager (Confirmed)

Path for demo tenant:

```
cci-lite/oak/clarifygo
```

Contains:

* base_url
* client_id
* client_secret
* username
* password
* token_path
* recordings_path
* recording_download_path

This is the template for all customer onboarding.

---

## 6. CloudWatch Logs (Confirmed)

Log groups exist for all Lambdas:

```
/aws/lambda/cci-lite-oak-ingest
/aws/lambda/cci-lite-job-init
/aws/lambda/cci-lite-result-handler
/aws/lambda/cci-lite-analyser
```

All Lambdas produce valid logs.

---

## 7. Tenant Strategy (Locked)

* A **single set of buckets** is shared by all customers.
* Each tenant uses **org_id prefix from Clerk**.
* No databases.
* No Supabase.
* All tenant-specific config lives in S3 and Secrets Manager.

---

# 🎯 Final Outcome — Phase 0 & 1 Complete

AWS backend + Clerk-only multi-tenant architecture is:

* Fully operational
* MVP-ready
* Safe to build frontend and backend on top of
* Infinitely extensible for future automation phases

This document is the **canonical reference** for CCI Lite MVP.

Next phase to be added to this file:

# ▶️ Phase 2 — Backend Architecture (Clerk + S3 API)

# ▶️ Phase 3 — Frontend Architecture & Routing

# ▶️ Phase 4 — Dashboard Implementation

# ▶️ Phase 5 — Tenant Onboarding Manual

# ▶️ Phase 6 — Internal QA & Testing

# ▶️ Phase 7 — Pilot Deployment

# ▶️ Phase 8 — Public Launch

(Updates will append to this README.md in order.)
