# CCI Lite – EventBridge Pipeline (Factual AWS Implementation)

This document describes **only the EventBridge rules that actually exist** in your AWS account according to the exported AWS configuration. No hypothetical or unused rules are included.

---

## 1. Overview

Amazon EventBridge is used minimally in CCI Lite. In your actual deployment, EventBridge serves **two** purposes:

1. **Scheduling the OAK ingest Lambda** (currently disabled)
2. **Handling Amazon Transcribe job completion events** (monitoring Lambda)

No other EventBridge rules were found in the export. Notably:

* There are **no S3 ObjectCreated → Lambda** rules.
* There are **no enriched-output triggers**.
* There are **no result-handler triggers**.

Your pipeline today is therefore only partially event‑driven.

---

## 2. EventBridge Rule: `cci-lite-oak-ingest-schedule`

**State:** DISABLED
**Purpose:** Would trigger the OAK ingest Lambda on a schedule.
**Schedule Expression:** `rate(60 minutes)`

### Target

* **Lambda:** `cci-lite-oak-ingest`

### Notes

* The rule exists but is currently disabled.
* When enabled, it would poll the OAK API hourly.

---

## 3. EventBridge Rule: `cci-lite-transcribe-complete`

**State:** ENABLED
**Purpose:** Handles **Amazon Transcribe job completion events**.

### Event Pattern (simplified)

The rule listens for:

* `source = aws.transcribe`
* `detail-type = ...State Change`
* `TranscriptionJobStatus = COMPLETED`

This is the only active event-driven link in the current backend.

### Target

* **Lambda:** `cci-lite-transcribe-monitor`

### Notes

* This Lambda is likely responsible for detecting when a Transcribe job has finished and performing follow‑up processing.
* No downstream EventBridge-based triggers were found for analyzers or result handlers.

---

## 4. What EventBridge Is *Not* Doing (Factual)

Based on your AWS export, EventBridge is **not** currently wired to:

* Trigger the analyzer when audio/meta is uploaded
* Trigger the result-handler when enriched output appears
* Dispatch any S3 notifications
* Handle pipeline orchestration between components

If the analyzer and result-handler are executing today, they are triggered by:

* Direct Lambda invocation
* Code‑level orchestration
* Manual testing
* Or unexported mechanisms outside EventBridge

This is an important architectural gap to address later.

---

## 5. Summary

EventBridge usage in the current environment is minimal and limited to:

| Rule Name                      | Enabled | Purpose                                 | Target Lambda                 |
| ------------------------------ | ------- | --------------------------------------- | ----------------------------- |
| `cci-lite-oak-ingest-schedule` | No      | Scheduled OAK polling (disabled)        | `cci-lite-oak-ingest`         |
| `cci-lite-transcribe-complete` | Yes     | Handle Transcribe job completion events | `cci-lite-transcribe-monitor` |

No other rules exist.

This document now reflects the **actual, factual** EventBridge configuration with no assumptions, no hypotheticals, and no future-state scenarios.

---

**Next:** `s3-structure.md`
