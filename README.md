# CCI Lite MVP Build — Phase 0 & Phase 1 Documentation

This document captures the fully locked and validated Phase 0 and Phase 1 architecture for the CCI Lite MVP. It is intended for version control in GitHub and will act as the foundation for all subsequent build phases.

---

## 🚀 Overview

CCI Lite MVP follows a lean multi‑tenant architecture using:

* **Clerk** for authentication only
* **Supabase** for tenant/user mapping (manual provisioning)
* **AWS S3 + Lambda + EventBridge + Secrets Manager + Transcribe + Bedrock + Comprehend** as the processing pipeline

This document outlines:

* Phase 0 — Foundation Lock‑In
* Phase 1 — AWS Infrastructure Confirmation

Everything below is now locked and authoritative.

---

# ⭐ PHASE 0 — Foundation Lock‑In

## ✅ Deliverables

* Architecture locked
* Tenancy model locked (manual onboarding)
* Data flow locked
* Dashboard requirements frozen

## ✅ Actions Confirmed

### 1. Clerk = Authentication Only

Clerk session token includes only the required claims:

```json
{
  "org_id": "{{organization.id}}",
  "org_name": "{{organization.name}}",
  "user_id": "{{user.id}}",
  "email": "{{user.primary_email_address}}"
}
```

No webhooks.
No provisioning logic.
No automation.

### 2. Supabase = Tenant/User Mapping Only

Supabase stores identity and role information.
Tables:

* **tenants**
* **users**
* **subtenants** (optional, future use)

No call data is stored in Supabase.
All call data lives in S3.

### 3. AWS = Full Processing Environment

All ingestion, transcription, enrichment, and AI analysis happens in AWS.
Backend/frontend must *read from AWS*, not process locally.

### 4. Tenant Provisioning = Manual

For MVP:

* No automated S3 folder creation
* No automated Supabase tenant creation
* No automated Secrets Manager provisioning

### 5. S3 Tenant Prefixes = Manual Creation

Example structure:

```
<bucket>/tenant-id/...folders...
```

Each customer/tenant gets a manually created prefix.

### 6. Secrets Manager = Manual Per-Tenant Provider Keys

Example secret path:

```
cci-lite/oak/<tenant>
```

Stored values:

* base_url
* client_id
* client_secret
* username
* password
* token_path
* recordings_path
* recording_download_path

### 7. Data Flow Locked

End-to-end pipeline:

```
Recording API → ingestion → S3 audio/meta → Transcribe → raw JSON → enrichment → analyzer → final JSON → dashboard
```

### 8. Dashboard MVP Requirements Locked

Sections:

* Business Overview
* QA Overview
* Agent Performance
* Calls Explorer
* Call Detail Page

All powered **only** by the JSON/v2.0-flat outputs.

**Phase 0 Status: COMPLETE & LOCKED**

---

# ⭐ PHASE 1 — AWS Infrastructure Confirmation

This phase validates that AWS backend is fully aligned with the MVP architecture and functioning correctly.

## 1. S3 Buckets (Confirmed)

```
cci-lite-config-591338347562-eu-central-1
cci-lite-input-591338347562-eu-central-1
cci-lite-results-591338347562-eu-central-1
```

Region: **eu-central-1 (Frankfurt)**

### Input Bucket Structure

```
demo-tenant/
    audio/
    meta/
```

### Results Bucket Structure

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

### Config Bucket Structure

```
demo-tenant/
    ai-config.json
    qa-config.json
```

## 2. Lambda Functions (Confirmed)

Functions deployed:

```
cci-lite-oak-ingest
cci-lite-job-init
cci-lite-result-handler
cci-lite-analyser
```

All functions validated, correct runtime, environment variables, and bucket paths.

### Function Summary

#### cci-lite-oak-ingest

* Python 3.14
* Pulls recordings from ClarifyGo
* Writes into `input/demo-tenant/audio/` and `meta/`
* Scheduled via EventBridge (Step 4)

#### cci-lite-job-init

* Python 3.13
* Trigger: S3 `ObjectCreated:Put` on audio/
* Sends audio to Transcribe
* Stores raw transcripts in `results/demo-tenant/raw/`

#### cci-lite-result-handler

* Trigger: S3 `ObjectCreated` on raw/
* Enrichment via Comprehend
* Writes to `enriched/`

#### cci-lite-analyser

* Trigger: SQS event mapping
* Runs Bedrock + Comprehend
* Writes final outputs:

  * `final/v1.0/`
  * `final/v2.0-flat/calls/`
  * `final/v2.0-flat/qa/`

## 3. EventBridge Rules (Confirmed)

* `cci-lite-oak-ingest-schedule`
* `rate(15 minutes)` for MVP
* Correct permissions added automatically

## 4. IAM Role Validation (Complete)

Attached policies include:

* AmazonTranscribeFullAccess
* AWSLambdaBasicExecutionRole
* cci-lite-analyzer-policy (inline)
* cci-lite-lambda-policy (inline)
* CCI-Lite-OAKIngestPermissions (inline)
* sqs-trigger-policy (inline)

All required AWS services: S3, Transcribe, Comprehend, Secrets Manager, Bedrock, SQS.

**IAM is correct for MVP.**

## 5. Secrets Manager (Confirmed)

Secret path:

```
cci-lite/oak/clarifygo
```

Contains all ClarifyGo API values.

This will be replicated per tenant during onboarding.

## 6. CloudWatch Logs (Confirmed)

Log groups present:

```
/aws/lambda/cci-lite-oak-ingest
/aws/lambda/cci-lite-job-init
/aws/lambda/cci-lite-result-handler
/aws/lambda/cci-lite-analyser
```

All Lambdas are logging correctly.

## 7. Tenant Strategy

* Single set of buckets
* Multi-tenant via prefixes (PERFECT for MVP)
* demo-tenant is canonical test tenant

## 8. Final Outcome

AWS backend is fully operational and production-ready for MVP.

**Phase 1 Status: COMPLETE & LOCKED**

---

# 📌 NEXT STEPS

Proceed to Phase 2 (Backend architecture + Clerk + Supabase wiring) once this document is committed to GitHub.

This file serves as the permanent record of the foundation and AWS alignment for CCI Lite MVP.
