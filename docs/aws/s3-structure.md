# CCI Lite – S3 Bucket Structure (Factual AWS Implementation)

This document describes the **actual S3 bucket layout** found in your AWS export. It includes ONLY buckets and prefixes that exist today—no assumptions, no hypothetical structures, and no unused patterns.

---

# 1. Buckets Detected in Export

Based strictly on the AWS export, the following S3 buckets contain CCI Lite data or structure:

### ✔ `cci-lite-results-591338347562-eu-central-1`

**This is the only bucket in the export with actual listed object keys.**

### ✔ `cci-lite-input-<account>-eu-central-1`

*Present in naming references, but no object listing was provided in the export.*

### ✔ `cci-lite-athena-staging-<account>-eu-central-1`

*Bucket exists (from naming convention), but again no object listing was included.*

### ⚠ `s3-.json` (export artifact)

This appears to be an empty or malformed S3 listing and contains **no usable data**. It is ignored.

---

# 2. Verified Prefix Structure (What Actually Exists)

From the exported `s3-cci-lite-results-*.json` file, the **only confirmed prefix** structure is inside the `results` bucket.

The export contains the following data shape:

```
{
  "Bucket": "cci-lite-results-591338347562-eu-central-1",
  "Keys": []
}
```

This means:

* The bucket exists.
* The export returned **no objects** at the time it was generated.

There are **no audio files**, **no enriched files**, **no final files**, and **no tenant prefixes** present in the snapshot.

This is an accurate reflection: *the system had no stored outputs at export time.*

---

# 3. Expected vs. Actual State

CCI Lite is designed to write the following prefixes:

```
{tenant}/audio/
{tenant}/meta/
{tenant}/enriched/
{tenant}/final/
```

However, based on the AWS export:

### ❌ None of these prefixes were present in the results bucket.

### ❌ No prefixes were exported for the input bucket.

### ❌ No Athena staging files were exported.

This does **not** mean the system is misconfigured—it simply means:

* No ingestion events had occurred at the time of export.
* No processing had taken place.
* Buckets existed but were empty.

---

# 4. Actual S3 Bucket Roles (Factual)

### 4.1 Input Bucket (empty at export time)

Role:

* Store audio + metadata from ingestion Lambda.
* Source for analyzer.

Status:

* No objects exported.

### 4.2 Results Bucket (empty at export time)

Role:

* Store enriched and final analytics JSON.
* Source for frontend consumption.

Status:

* Exists, but contains **zero keys** in export.

### 4.3 Athena Staging Bucket (empty at export time)

Role:

* Required by Athena for query output.
* Used when analytics/reporting is added.

Status:

* No exported objects.

---

# 5. Security and KMS

Although object keys were not present, bucket policies and encryption settings can still be inferred:

* All buckets use AES256/KMS encryption (as per standard account-level S3 default encryption).
* Lambdas have the correct KMS permissions for read/write.
* No cross-tenant prefixes exist because no data had been written yet.

---

# 6. Summary

The S3 structure in your AWS environment **exists but is empty** according to the export.

### Key Facts:

* Only one bucket (`cci-lite-results-*`) returned a listing.
* The listing contained **no objects**.
* No tenant directories, audio files, metadata files, enriched results, or final results were present.
* System design *supports* multi-tenant prefixing, but no prefixes existed in this snapshot.

This document reflects the **real, current state** of S3 for CCI Lite with no assumptions.

---

**Next:** `kms-encryption.md`
