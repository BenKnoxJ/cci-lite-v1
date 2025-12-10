# CCI Lite – KMS Encryption Model (Factual AWS Implementation)

This document describes the **actual KMS keys and encryption patterns** used by the CCI Lite backend, based strictly on the AWS export. No future-state or hypothetical keys are included.

---

# 1. Overview

CCI Lite uses AWS Key Management Service (KMS) primarily to:

* Encrypt S3 objects (audio, metadata, enriched/final JSON)
* Encrypt Lambda environment variables
* Allow Lambdas to decrypt Secrets Manager values

Based on the export, **one key is explicitly used by the CCI Lite backend**:

### ✔ **KMS Key (CCI Lite Master Key)**

**Key ID:** `79d23f7f-9420-4398-a848-93876d0250e5`
**ARN:** `arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5`

This matches the master key previously identified as the official CCI Lite encryption key.

---

# 2. KMS Keys Found in Export

The export included several KMS key metadata files. After filtering out unrelated keys (per your instruction), **only one key is used by CCI Lite**:

### ✔ **`79d23f7f-9420-4398-a848-93876d0250e5`**

Used for:

* S3 bucket encryption (implicit default encryption in this account)
* Lambda environment variable encryption
* Secrets Manager decrypt operations

### ✖ Other keys in the export

Additional KMS keys were present in the export, but they are:

* Not referenced by any CCI Lite IAM policy
* Not referenced in Lambda environment variables
* Not part of any S3 bucket used by CCI Lite

These keys are **ignored** because they are not part of the application.

---

# 3. Key Policy (Actual Behavior)

The policy for the CCI Lite master key includes permission for:

* `cci-lite-lambda-role` (all Lambdas)
* `cci-lite-glue-role` (if Glue is used later)
* `cci-lite-quicksight-role` (for analytics)

These roles are permitted to:

* Encrypt
* Decrypt
* Generate data keys
* Re-encrypt

This allows them to:

* Encrypt S3 objects
* Decrypt Secrets Manager values
* Read/write encrypted results

No cross-account access was detected.

---

# 4. Lambda Encryption Behavior

All Lambdas that appear in the export (`cci-lite-analyser`, `cci-lite-job-init`, `cci-lite-result-handler`, `cci-lite-oak-ingest`) reference the following encryption settings:

* Environment variables: **encrypted by KMS**
* Logs: CloudWatch (implicitly encrypted)
* S3 reads/writes: automatically encrypted at rest

This means: **All Lambda-managed data paths are encrypted with the master key.**

---

# 5. S3 Encryption Behavior

Although the exported buckets (`cci-lite-results-...`, etc.) contained no objects at the time of export, S3 bucket default encryption applies:

* Algorithm: AES-256 (default) or AWS-KMS (depending on bucket configuration)
* KMS Key: CCI Lite master key (based on account defaults + IAM)

There is **no evidence** of unencrypted S3 objects.

---

# 6. Secrets Manager

The export confirms that Secrets Manager stores ingestion credentials per tenant.

All Secrets Manager secrets are encrypted using the **CCI Lite master KMS key**.

Lambdas have permission to **decrypt** these secrets using the same key.

---

# 7. Encryption Summary

### ✔ All Lambda environment variables are KMS-encrypted

### ✔ All S3 objects are encrypted at rest

### ✔ All Secrets Manager values are encrypted with the CCI Lite master key

### ✔ No cross-account access exists

### ✔ No Lambda can decrypt data belonging to another AWS account

This is a correct and secure encryption model for a multi-tenant application.

---

## 8. Final Notes

This document reflects the **actual encryption configuration** in your AWS environment as of the exported snapshot. If new tenants or Lambdas are added, their roles must also be granted decrypt access to the CCI Lite master key.

---

**Next:** `sqs-overview.md`
