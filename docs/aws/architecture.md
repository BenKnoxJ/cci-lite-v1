# CCI Lite – AWS Architecture Overview

This document provides a complete, high-level architectural description of the CCI Lite backend as reconstructed from the AWS resource export. It is the authoritative reference for how the CCI Lite backend is structured, how data flows through the system, and how multi-tenant behavior is enforced.

---

## 1. Architectural Purpose

CCI Lite is a multi-tenant, event-driven, serverless analytics pipeline built on AWS. Its primary goals are:

* Ingest call recordings and metadata from external systems
* Process and enrich recordings using AI (Amazon Bedrock)
* Produce structured results for downstream reporting and dashboards
* Enforce strict tenant isolation at every stage

The system is designed for simplicity, determinism, low operational overhead, and secure handling of customer audio and metadata.

---

## 2. High-Level Architecture Summary

The backend is composed of four major subsystems:

### **2.1 Ingestion Subsystem**

Responsible for pulling audio and metadata from external sources (e.g., OAK, 3CX, ASC).

**Key Components:**

* `cci-lite-oak-ingest` Lambda
* Secrets Manager (tenant credentials)
* S3 Input Bucket (`cci-lite-input-<account>-eu-central-1`)

**Outputs:**

* `{tenant}/audio/...`
* `{tenant}/meta/...`

Ingestion writes raw audio and call metadata into tenant-scoped S3 prefixes.

---

### **2.2 Analysis Subsystem**

Triggered when new audio/meta files arrive.

**Key Components:**

* EventBridge rule (S3 create → analysis trigger)
* `cci-lite-analyser` Lambda
* Amazon Bedrock (Claude Haiku)
* SQS queue (optional async mode)

**Function:**

* Fetches raw audio + meta
* Calls Bedrock to generate enriched JSON (schema v1.1)
* Writes enriched results into the results bucket:

  * `cci-lite-results-<account>/ {tenant}/enriched/...`

---

### **2.3 Result Handling Subsystem**

Finalises enriched results and prepares them for consumption.

**Key Components:**

* EventBridge rule → `cci-lite-result-handler` Lambda
* DynamoDB (`cci-lite-config`)
* S3 Final Output Folder

**Function:**

* Normalises AI output
* Performs clean-up or joins metadata
* Writes final result JSON to:

  * `{tenant}/final/...`

---

### **2.4 Consumption Subsystem**

Used by the frontend (Replit/Vercel Next.js application).

**Key Components:**

* S3 Results Bucket (read-only via signed URLs)
* Clerk authentication → tenant_id claim
* Next.js API routes enforcing prefix isolation

Frontend retrieves enriched/final JSON files filtered by tenant.

---

## 3. Core AWS Services Involved

| Service         | Purpose                                            |
| --------------- | -------------------------------------------------- |
| Lambda          | Compute for ingestion, analysis, and finalisation  |
| S3              | Storage for audio, metadata, and result JSONs      |
| EventBridge     | Triggers analysis and finalisation workflows       |
| Bedrock         | AI enrichment (Claude Haiku)                       |
| SQS             | Optional buffering for analyzer workloads          |
| Secrets Manager | Stores per-tenant ingestion credentials            |
| DynamoDB        | Stores per-tenant configuration                    |
| KMS             | Encrypts S3 objects and Lambda environment secrets |
| CloudWatch      | Logs for Lambdas and operational monitoring        |

---

## 4. Multi-Tenant Architecture Model

CCI Lite enforces tenant isolation using:

### **4.1 S3 Prefix Isolation**

Every object is stored under:

```
{tenant}/audio/
{tenant}/meta/
{tenant}/enriched/
{tenant}/final/
```

Tenants cannot access each other's objects because the application and Lambda roles restrict access to only their own prefixes.

### **4.2 Secrets Manager Namespacing**

Each tenant has its own set of credentials or ingestion settings stored in unique secrets.

### **4.3 Role-Based Isolation**

Lambda roles permit only specific S3 prefixes and limited KMS decrypt scope.

### **4.4 Application-Level Isolation (Frontend)**

Clerk → JWT → tenant_id mapping ensures all reads are restricted to a single tenant.

---

## 5. Data Flow Overview (End-to-End)

Below is the conceptual data lifecycle:

1. **Ingestion:** External provider → `cci-lite-oak-ingest` → S3 `{tenant}/audio` + `{tenant}/meta`.
2. **Trigger:** S3 event → EventBridge rule.
3. **Analysis:** `cci-lite-analyser` → Bedrock → writes `{tenant}/enriched` JSON.
4. **Finalisation:** EventBridge rule → `cci-lite-result-handler` → writes `{tenant}/final`.
5. **Consumption:** Frontend retrieves final JSONs via signed URLs.

This flow is fully serverless and event-driven.

---

## 6. Security & Encryption Model

* **KMS:** All S3 buckets and Lambda environment variables use KMS-managed encryption.
* **IAM:** Lambda roles use least-privilege access patterns.
* **Data at Rest:** Encrypted in S3 and DynamoDB.
* **Data in Transit:** HTTPS enforced globally.
* **Secrets:** Stored exclusively in Secrets Manager; never in environment variables.

Tenant isolation depends on strict S3 prefix scoping and IAM boundaries.

---

## 7. Reliability & Scalability

The architecture is serverless and scales automatically:

* Lambdas scale with traffic volume
* SQS enables decoupling (optional)
* S3 is fully elastic
* EventBridge reliably triggers ingestion/analysis processes

No servers or containers need to be maintained.

---

## 8. Summary

CCI Lite’s backend is a clean, event-driven, multi-tenant, serverless pipeline. It leverages AWS-native services to:

* Securely ingest audio and metadata
* Enrich content with AI
* Store structured analytics outputs
* Provide them to the frontend with strict tenant isolation

This document acts as the foundation for deeper documentation in subsequent files, including IAM, Lambda behaviors, data lifecycle maps, and security analysis.

---

**Next Document:** `lambda-overview.md`
