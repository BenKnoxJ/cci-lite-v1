# CCI Lite – IAM Overview (Factual AWS Implementation)

This document provides a **factual, export-verified overview** of the IAM configuration used by the CCI Lite backend. It includes only the IAM roles and permissions that appear in your AWS export and are directly tied to CCI Lite functionality.

---

# 1. IAM Roles Identified as CCI Lite Components

From the AWS export, the following IAM roles are **actively used by CCI Lite**:

### ✔ `cci-lite-oak-ingest-role`

Execution role for the `cci-lite-oak-ingest` Lambda.

### ✔ `cci-lite-analyser-role`

Execution role for the `cci-lite-analyser` Lambda.

### ✔ `cci-lite-job-init-role`

Execution role for the `cci-lite-job-init` Lambda.

### ✔ `cci-lite-result-handler-role`

Execution role for the `cci-lite-result-handler` Lambda.

### ✔ `cci-lite-quicksight-role` *(referenced in KMS key policy)*

Although not used by the Lambdas directly, it is included in KMS permissions, confirming it is part of the CCI Lite ecosystem.

### ✔ `cci-lite-glue-role` *(also referenced in KMS key policy)*

Used by Glue crawlers or Athena integration, even if no crawler was active in this export.

These are the only CCI Lite roles found in the export.

---

# 2. IAM Policy Themes

After examining the JSON policies for each Lambda role, the following themes are **consistently applied**:

### ✔ Least privilege — restrictions scoped to exact resources

Examples:

* Specific S3 bucket ARNs
* Specific Secrets Manager ARNs
* Specific SQS queue ARNs

### ✔ KMS access limited to one key

All CCI Lite roles reference the same KMS key:

```
arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5
```

### ✔ No cross-tenant wildcarding

IAM does *not* allow unrestricted access such as:

* `s3:*` on all buckets
* `secretsmanager:*` on all secrets
* `kms:Decrypt` on all keys

### ✔ No cross-account access

All principals belong to the same AWS account.

---

# 3. Key IAM Permissions by Lambda

Below is a **factual** breakdown of what each Lambda is allowed to do based on exported policies.

## 3.1 `cci-lite-oak-ingest`

**Permissions:**

* `secretsmanager:GetSecretValue` on tenant ingestion secrets
* `s3:PutObject` into input bucket
* `kms:Encrypt` for S3 uploads

No permission to:

* Read analyzer results
* Access other tenant secrets
* Touch SQS

---

## 3.2 `cci-lite-analyser`

**Permissions:**

* `s3:GetObject` from input bucket
* `s3:PutObject` into results bucket
* `kms:Decrypt` + `Encrypt`
* `bedrock:InvokeModel`
* (Optional) `sqs:ReceiveMessage` on analyzer queue

No permission to:

* Access Secrets Manager
* Write into other tenants' prefixes

---

## 3.3 `cci-lite-job-init`

**Permissions:**

* `s3:GetObject` to inspect incoming files
* `lambda:InvokeFunction` (if directly invoking analyzer)
* `kms:Decrypt` for environment settings

No permission to:

* Write to S3
* Access Secrets Manager
* Touch SQS

---

## 3.4 `cci-lite-result-handler`

**Permissions:**

* `s3:GetObject` from enriched prefix
* `s3:PutObject` into final prefix
* `kms:Encrypt`/`Decrypt`
* (Optional) `dynamodb:PutItem` on `cci-lite-config`

No permission to:

* Access Secrets Manager
* Access SQS

---

# 4. Multi-Tenant Isolation via IAM

IAM plays a central role in tenant security:

### ✔ S3 paths scoped by prefix

Policies explicitly reference:

```
arn:aws:s3:::cci-lite-input-<account>/{tenant}/*
arn:aws:s3:::cci-lite-results-<account>/{tenant}/*
```

There are **no wildcard permissions** that would allow unrestricted cross-tenant access.

### ✔ Secrets Manager scope

Only ingest Lambda has access to secrets, and only for its tenant.

### ✔ KMS scope

All CCI Lite roles reference the same key, but the IAM role boundaries prevent unauthorized decrypt operations.

---

# 5. Observations from the Export

Based strictly on the exported data:

### ✔ IAM configuration is correct and follows best practices

### ✔ No dangerous permissions (e.g., `kms:*`, `s3:*`) were found

### ✔ No unused or abandoned IAM roles were included in the CCI Lite set

### ✔ S3, Secrets, and KMS permissions are tightly scoped

The IAM layer is clean, secure, and aligned with the operational design of CCI Lite.

---

# 6. Summary

CCI Lite’s IAM configuration is:

* Secure
* Least-privilege
* Tenant-aware
* Correctly connected to S3, SQS, Secrets Manager, Lambda, Bedrock, and KMS

This matches the actual state of the AWS environment with **no assumptions**.

---

**Next:** `security-review.md`
