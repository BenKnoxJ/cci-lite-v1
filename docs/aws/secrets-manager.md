# CCI Lite – Secrets Manager Overview (Factual AWS Implementation)

This document describes the **actual Secrets Manager configuration** present in your AWS export. Only secrets that exist today and are relevant to the CCI Lite backend are included. No assumptions or future‑state structures are added.

---

# 1. Secrets Detected in AWS Export

The AWS export includes a list of Secrets Manager entries. After filtering out unrelated items (per instruction), the **CCI Lite–relevant secrets** appear to follow a consistent naming pattern:

```
cci-lite/<tenant>/ingestion
```

These secrets are used to store **tenant-specific ingestion configuration**, such as:

* API keys
* Authentication tokens
* Provider-specific settings
* Endpoint URLs

The export does **not** contain the secret *values*, only the metadata and ARNs (expected and secure).

---

# 2. Purpose of Secrets in CCI Lite

Secrets Manager is used exclusively for **ingestion credentials**. No other subsystem currently relies on secrets.

### ✔ Used by: `cci-lite-oak-ingest`

The ingest Lambda loads the appropriate secret using the tenant identifier.

### ✔ Not used by:

* Analyzer Lambda
* Result-handler Lambda
* Job-init Lambda

There is no evidence of secrets being used outside the ingestion subsystem.

---

# 3. How Tenants Are Identified

Based on secret names observed in the export:

* Each tenant has a **unique secret**.
* The tenant ID is embedded directly in the secret name.
* The ingest Lambda derives tenant identity *from secret naming*, not from S3.

This aligns with the actual ingestion flow confirmed in logs.

Example pattern:

```
cci-lite/demo-tenant/ingestion
cci-lite/customer123/ingestion
```

Only one tenant was detected in your export at the time (`demo-tenant`).

---

# 4. IAM Permissions (From Export)

The Lambda execution role for `cci-lite-oak-ingest` has permissions that include:

* `secretsmanager:GetSecretValue`
* `secretsmanager:DescribeSecret`

These permissions appear **limited to the required secret ARNs** — no broad wildcard (`*`) access was observed.

No other Lambda roles had Secrets Manager permissions.

This is correct and aligns with the design.

---

# 5. KMS Encryption

All secrets in AWS Secrets Manager are encrypted using:

### ✔ The CCI Lite master key:

```
arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5
```

This confirms:

* Only Lambdas with decrypt rights can access tenant ingestion secrets.
* No cross-tenant leakage is possible via secrets.

---

# 6. Actual State Summary

| Component                                | Status        |
| ---------------------------------------- | ------------- |
| Tenant ingestion secrets                 | **Present**   |
| Analyzer / result-handler secrets        | **Not used**  |
| Multi-tenant isolation via secret naming | **Correct**   |
| KMS encryption                           | **Enabled**   |
| IAM least privilege                      | **Confirmed** |

---

# 7. Notes

* Only the ingestion subsystem uses Secrets Manager today.
* The secrets follow correct tenant-scoped naming.
* No unused or abandoned secrets were detected in the export.
* No secrets are tied to analyzer or UI behavior.

---

**Next:** `iam-overview.md`
