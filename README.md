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

## ✅ 6. ## Phase 0 — Section 6: Dashboard Requirements (CCI Lite Demo Prototype)

### **Overview**

This section defines the full dashboard requirements for the **CCI Lite Dashboard Prototype**, based strictly on the flattened CSV schema produced by the current CCI Lite v1.3 engine. Nothing in this specification introduces fields or insights that do not already exist in the enriched JSON → flattened Athena view.

The goal of this dashboard is to provide a clean, enterprise-grade demonstration of what CCI Lite can surface for businesses, QA managers, contact centres, and customer experience leaders. This is a **prototype**, but must reflect production-grade structure, UX patterns, and multi-tenant behaviour.

---

## **6.1 Principles & Constraints**

1. **Data truth = Flattened CSV / Athena View Only**
   Every chart, metric, table, or search must use *only* columns present in the flattened CSV.

2. **No artificial or invented metrics**
   If the CSV does not contain it, the dashboard must not display it.

3. **Multi-tenant ready**

   * Dashboard loads *only* the tenant’s S3 data.
   * Clerk provides auth and tenant isolation.

4. **Dynamic QA & Metadata Support**
   Tenants may have:

   * Different QA rule counts
   * Different recording metadata schemas
   * Different categorisation / call IDs / labels

   The dashboard must dynamically render based on what exists.

5. **No backend heavy lifting**
   The app reads directly from S3 JSON outputs via signed URL or API passthrough.

---

## **6.2 Dashboard Architecture Overview**

The prototype consists of **three core dashboard modules**:

### **1. Business Overview Dashboard**

High-level estate-wide view of all processed calls within the date range.

Uses only CSV fields such as:

* call_id
* started_at / ended_at
* duration_seconds
* direction
* calling_party / called_party
* sentiment scores
* transcript_word_count
* qa scores (per rule)
* overall_qa_pass

### **2. Agent Explorer + Drill-Down**

Search → See agent performance → Drill into individual calls.

Driven by:

* normalised agent identifier (from calling_party matching tenant domain)
* sentiment
* qa_rule results
* call duration
* call volume

### **3. Call Detail Insight Page**

Deep dive: one call, full transcript, metrics, sentiment and QA breakdown.

Driven by:

* Transcript utterances
* Comprehend sentiment outputs
* QA evaluation JSON schema
* Metadata fields (call start, end, direction)

---

## **6.3 Business Overview Dashboard (Global)**

### **6.3.1 KPIs**

All KPI cards must come directly from CSV fields:

* **Total Calls**
* **Total Duration (HH:mm)**
* **Average Call Duration**
* **Average Agent Sentiment** (using sentiment_score)
* **% QA Passed** (qa_overall_pass boolean)
* **Inbound vs Outbound Ratio** (using direction)

### **6.3.2 Charts**

All charts must map to flat fields:

* **Daily Call Volume Trend** (count by started_at date)
* **Direction Breakdown** (Pie: inbound/outbound/internal)
* **Sentiment Distribution** (Bar: positive/neutral/negative score groupings)
* **QA Pass/Fail Trend** (Line or Column)

### **6.3.3 Tables**

* **Top Agents by Call Volume**
* **Top Agents by Positive Sentiment**
* **Agents With Most QA Failures**

All using only agent_id, sentiment, and QA pass/fail available in the CSV.

---

## **6.4 Agent Explorer Dashboard**

### **6.4.1 Agent Search / Listing**

With only CSV data, agent identification comes from:

* Normalised calling_party or called_party fields flagged as internal agent domain

List includes:

* Agent Name/ID
* Total Calls
* Avg Duration
* Avg Sentiment
* QA Pass Rate

### **6.4.2 Agent Detail Page**

Metrics:

* Total calls
* Avg sentiment score
* Avg call duration
* QA pass rate

Charts:

* Sentiment trend over time
* QA fail reasons (aggregated by rule)
* Call volume trend

Table:

* List of all calls for that agent (call_id, date, duration, sentiment, QA result)

---

## **6.5 Call Detail Page (Deep Dive)**

Full breakdown for a single call_id.

### **6.5.1 Metadata**

* call_id
* calling_party / called_party
* started_at, ended_at
* duration_seconds
* direction

### **6.5.2 Transcript Viewer**

Data fields:

* speaker
* text
* timestamps

### **6.5.3 Sentiment**

* overall sentiment label
* positive/neutral/negative scores

### **6.5.4 QA Breakdown**

Dynamic:

* QA rule name
* Pass/Fail
* Reason/extract from the QA model (qa_rule_description)

### **6.5.5 Word Count & Utterance Metrics**

* transcript_word_count
* utterance_count
* agent vs customer talk ratio

---

## **6.6 Dynamic QA Rule Rendering**

Tenants will have different QA rules.
The dashboard must:

* Read the QA rules from each call record (flat CSV columns)
* Dynamically map each rule into the agent or call-level QA view
* If a rule does not appear in a tenant’s dataset, the front-end hides it automatically

This removes the need for hardcoded UI logic.

---

## **6.7 Multi-Tenant Normalisation Layer (Front-End)**

Even though CCI Lite is ingestion-agnostic (ClarifyGo, 3CX, etc.), the dashboard must standardise:

### **6.7.1 Agent Identification**

Rules:

* If calling_party endswith tenant domain → agent
* If neither end matches → unknown

### **6.7.2 Direction Mapping**

Based on logic inside the analyzer Lambda:

* inbound
* outbound
* internal
* unknown

### **6.7.3 Metadata Flexibility**

If a tenant ingestion source includes extra metadata fields, the dashboard should:

* Ignore unsupported columns
* Render extended metadata only if present

---

## **6.8 Data Loading Pattern**

The dashboard loads data using:

* Direct S3 GET → signed URL → JSON/CSV fetch
  or
* Lightweight API proxy that serves S3 objects

No computation occurs server-side.

---

## **6.9 Dashboard Pages Summary**

### **1. Login / Tenant Selection (Clerk)**

Tenant-isolated login.

### **2. Business Overview**

Estate-wide KPIs, charts, trends.

### **3. Agent Explorer**

Search agents → View agent metrics → Drill into calls.

### **4. Agent Detail Page**

Charts, QA summary, sentiment, call list.

### **5. Call Detail Page**

Full transcript, QA, sentiment, metadata.

---

## **6.10 What This Prototype Demonstrates**

This dashboard acts as a powerful demonstration layer that shows:

* What businesses will be able to see
* How QA and sentiment insights surface operational problems
* How call intelligence reveals customer experience patterns
* How multi-tenant data can be standardised and visualised elegantly

It is intentionally simple on backend, powerful on UX, and designed for rapid iteration into a full CCI SaaS product.


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
