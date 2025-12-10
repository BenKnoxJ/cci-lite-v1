# CCI Lite – Lambda Functions Overview

This document provides a complete technical overview of the Lambda functions that power the CCI Lite backend. Each Lambda is described in terms of purpose, triggers, input/output behavior, IAM permissions, multi-tenant considerations, and integration points.

---

## 1. Overview of Lambda Functions

CCI Lite uses four core Lambda functions:

1. **`cci-lite-oak-ingest`** — Ingests audio and metadata from external providers.
2. **`cci-lite-analyser`** — Processes raw audio + metadata and generates enriched AI output.
3. **`cci-lite-job-init`** — Pre-processing or orchestration logic triggered by S3/EventBridge.
4. **`cci-lite-result-handler`** — Finalises enriched results and writes final JSON outputs.

Each plays a role in the end-to-end post-call analytics flow.

---

# 2. `cci-lite-oak-ingest`

### **Purpose**

Collects call recordings and metadata from an external provider (e.g., OAK) and stores them in S3 under tenant-specific prefixes.

### **Triggers**

* EventBridge schedule (periodic polling)
* Manual invocation (for debugging/testing)

### **Inputs**

* Provider API response containing: audio file URL, metadata JSON
* Secrets Manager tenant credentials

### **Outputs (S3)**

```
s3://cci-lite-input/{tenant}/audio/{call_id}.wav
s3://cci-lite-input/{tenant}/meta/{call_id}.json
```

### **IAM Permissions**

* Read: Secrets Manager (tenant credentials)
* Write: S3 input bucket (tenant prefixes only)
* KMS: Encrypt objects using the CCI Lite master key

### **Multi-Tenant Behavior**

* Tenant ID is derived from secret naming
* Ingestion never writes outside its assigned tenant prefix

---

# 3. `cci-lite-analyser`

### **Purpose**

Transforms raw audio + metadata into structured analytics using Amazon Bedrock.

### **Triggers**

* EventBridge (S3 ObjectCreated events)
* Optional: SQS message (async mode)

### **Inputs**

* S3 audio + meta JSON from `{tenant}/audio` and `{tenant}/meta`
* AI configuration (hardcoded or S3-configured)

### **Processing Steps**

1. Fetch raw audio and metadata
2. Call Bedrock (Claude Haiku) with structured prompts
3. Validate AI output against schema v1.1
4. Generate enriched JSON object (sentiment, scores, categories, etc.)

### **Outputs (S3)**

```
s3://cci-lite-results/{tenant}/enriched/{call_id}.json
```

### **IAM Permissions**

* Read: S3 input bucket tenant prefix
* Write: S3 results bucket tenant prefix
* Read: Secrets Manager (optional)
* Encrypt/Decrypt: KMS master key
* Call: Bedrock API actions

### **Multi-Tenant Behavior**

* Tenant ID extracted from S3 key path
* Analyzer cannot read/write cross-tenant data due to IAM + code validation

---

# 4. `cci-lite-job-init`

### **Purpose**

Handles pre-processing tasks before analysis begins.

### **Common Functions**

* Validate that meta/audio pairs match
* Trigger analyzer workflow based on object type
* Clean up invalid or partial uploads

### **Triggers**

* EventBridge (ObjectCreated notifications)

### **Outputs**

* Internal events or direct invocation of analyzer Lambda

### **IAM Permissions**

* Read S3 input bucket
* Write to EventBridge (if dispatching events)
* Use KMS for decryption of environment variables

---

# 5. `cci-lite-result-handler`

### **Purpose**

Finalises enriched AI results and stores them as consumable JSON for the frontend.

### **Triggers**

* EventBridge after analyzer completion

### **Inputs**

* Enriched JSON from `{tenant}/enriched/{call_id}.json`

### **Processing Steps**

1. Load enriched AI-generated JSON
2. Standardise structure (normalisation)
3. Perform data cleanup or QA steps
4. Save final JSON document

### **Outputs (S3)**

```
s3://cci-lite-results/{tenant}/final/{call_id}.json
```

### **IAM Permissions**

* Read enriched S3 prefixes
* Write final S3 prefixes
* Optional: Update DynamoDB (`cci-lite-config` table)
* Use KMS for encryption

### **Multi-Tenant Behavior**

* Result handler derives tenant from the incoming S3 key
* IAM policies ensure output is stored in the correct tenant prefix

---

# 6. Lambda-to-Lambda and Lambda-to-Service Integrations

| Source Lambda    | Destination    | Purpose                     |
| ---------------- | -------------- | --------------------------- |
| oak-ingest       | S3 Input       | Store raw audio & metadata  |
| S3 + EventBridge | job-init       | Begin processing workflow   |
| job-init         | analyser       | Kick off Bedrock enrichment |
| analyser         | S3 Results     | Write enriched analytics    |
| analyser         | SQS (optional) | Async queueing for scale    |
| EventBridge      | result-handler | Finalise results            |

---

# 7. Cross-Lambda Shared Characteristics

### **Shared Environment Variables**

Typical values:

* AWS region
* Input bucket name
* Results bucket name
* KMS key ARN
* Optional Bedrock model configuration

### **Shared Security Model**

* All Lambdas operate with least-privilege IAM
* All handle data encrypted with KMS
* No Lambda has permission to access another tenant’s data

---

# 8. Summary

The Lambda layer in CCI Lite forms the backbone of the post-call analytics pipeline. Each function has a narrow, well-defined purpose, enabling clear scalability, simple debugging, and tightly enforced security boundaries.

This document pairs with:

* `eventbridge-pipeline.md`
* `iam-overview.md`
* `s3-structure.md`
