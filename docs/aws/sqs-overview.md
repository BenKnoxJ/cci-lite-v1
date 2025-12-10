# CCI Lite – SQS Overview (Factual AWS Implementation)

This document describes the **actual SQS queues and configurations** found in your AWS export. It includes only the queues that exist today and play a role in the current backend.

---

# 1. Queues Identified in AWS Export

Based strictly on the exported AWS configuration, **two** SQS queues are present and relevant to CCI Lite:

### ✔ `cci-lite-analyzer-queue`

Main analyzer queue used for asynchronous invocation of the analyzer Lambda.

### ✔ `cci-lite-analyzer-dlq`

Dead-letter queue associated with the analyzer queue.

No other SQS queues were present in the export.

---

# 2. `cci-lite-analyzer-queue`

### **Purpose**

Enables asynchronous processing for the analyzer Lambda (`cci-lite-analyser`). This decouples ingestion speed from analysis speed and provides buffering during spikes in audio ingestion.

### **Status From Export**

The queue exists with a valid attribute set. Relevant attributes observed:

* Visibility timeout – present
* Message retention – configured
* Redrive policy – enabled (points to DLQ)

### **Usage**

Although the queue exists, the EventBridge export did **not** show it being used as an EventBridge target. This implies that the queue is likely invoked:

* Manually from code, *or*
* Via direct Lambda-to-SQS integration

No triggers were exported for this queue.

---

# 3. `cci-lite-analyzer-dlq`

### **Purpose**

Stores failed analyzer messages after the maximum retry count is exceeded.

### **Status**

The DLQ exists and is correctly linked to the main analyzer queue.

This ensures failed analysis jobs do not disappear silently.

### **Contents at Export Time**

The export does **not** include messages, so its occupancy cannot be determined.

---

# 4. Queue Triggering (Factual)

The AWS export shows **no SQS → Lambda trigger mappings**.

This means:

* SQS is not configured to invoke `cci-lite-analyser` automatically.
* Processing must occur via Lambda-initiated polling or manual dispatch from another component.

There are **no** EventBridge rules delivering messages into SQS.

This document therefore reflects the factual state:

> SQS exists but is not wired into an automated event-driven path.

---

# 5. IAM Access

Although IAM policies were not printed here, the export confirms:

* Analyzer Lambda has `sqs:ReceiveMessage` and `sqs:DeleteMessage` permissions.
* DLQ access is implicitly granted for monitoring.

No other Lambdas were observed to have SQS permissions.

---

# 6. Summary

Based on the AWS export:

| Queue Name                | Purpose                      | Triggered? | Notes                          |
| ------------------------- | ---------------------------- | ---------- | ------------------------------ |
| `cci-lite-analyzer-queue` | Async analyzer processing    | No         | Exists and configured with DLQ |
| `cci-lite-analyzer-dlq`   | Capture failed analyzer jobs | N/A        | Proper redrive linkage exists  |

### Key Takeaways

* Only the analyzer subsystem uses SQS today.
* No automatic SQS → Lambda triggers are configured.
* The DLQ is properly linked.
* The queues are present and valid, but not actively integrated into event-driven pipeline execution.

This is the complete and factual representation of SQS usage in the CCI Lite backend.

---

**Next:** `dynamodb-schema.md`
