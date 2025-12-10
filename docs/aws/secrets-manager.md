# CCI Lite – Secrets Manager Overview (Factual AWS Implementation)

This document describes the **actual Secrets Manager configuration** present in your AWS export. Only secrets that exist today and are relevant to the CCI Lite backend are included. No assumptions or future-state structures are added.

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

There is no evidence of secrets being used outsid
