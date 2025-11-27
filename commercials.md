# Whisper v3 GPU Transcription Architecture — Costing, Scaling & Commercial Model

This document consolidates all validated calculations, scenarios, architectural assumptions, throughput benchmarks, and commercial outputs for deploying Whisper v3 (medium.en INT8) on AWS EC2 GPU (g5.xlarge) using a scale-to-zero SQS-driven transcription worker.

It is designed as a long‑form reference for Conversant CCI Lite/Premium engineering, commercial modelling, finance, and future planning.

---

## 1. Overview

Whisper v3 running on GPU-backed EC2 delivers extremely high throughput at very low cost when optimised correctly (concurrency, INT8 quantisation, medium.en model, streaming architecture). The GPU becomes a high‑throughput multi‑tasking engine capable of processing thousands of hours of audio per month.

The architecture runs:

* **EC2 g5.xlarge (A10G 24GB GPU)**
* **SQS → GPU Worker (Whisper) → S3 Transcript → Analyzer → Bedrock**
* **Scale-to-zero** so GPU only costs money when transcription is required

This document provides the:

* Throughput benchmarks (worst, real, best)
* Costs
* Scaling characteristics
* Agent capacity
* Per-agent cost to Conversant
* Pricing & profitability guidance

---

## 2. GPU Specification (Base Assumption)

**Instance:** g5.xlarge (A10G 24GB VRAM)
**Region:** eu-central-1
**Cost:** ~£0.74/hour → **£532.8/month** at 720 hours
**Total all-in cost (Bedrock, S3, Lambda, Logs):** ~**£550/month**

This £550/month is treated as the **maximum cost per GPU**.

---

## 3. Whisper v3 Performance Profiles

Optimised for:

* Model: **medium.en INT8**
* Concurrency: **8–12 streams**
* Speed: **3–6× real time**

The three validated throughput tiers:

### **Optimised Worst Case**

* Speed: **3× real time**
* Concurrency: **8 streams**
* Throughput: **24 audio-hours per GPU-hour**
* Monthly capacity: **17,280 hours**
* Cost per audio-hour: **£0.03**

### **Real‑World Typical**

* Speed: **4–5× real time**
* Concurrency: **10–12 streams**
* Throughput: **40–60 audio-hours per GPU-hour**
* Monthly capacity: **28,800–36,000 hours**
* Cost per audio-hour: **£0.015–£0.019**

### **Optimised Best Case**

* Speed: **6× real time**
* Concurrency: **12 streams**
* Throughput: **72 audio-hours per GPU-hour**
* Monthly capacity: **51,840 hours**
* Cost per audio-hour: **£0.010–£0.011**

---

## 4. Agent Recording Behaviour & FUP

To accurately model per-agent cost, the document uses these validated recording patterns:

* **Light user:** 20 hrs/month
* **Normal user:** 40–60 hrs/month
* **Heavy contact-centre user:** 80–100 hrs/month
* **Extreme outlier:** 120 hrs/month

**120 hours/month** is used as the **commercial fair usage upper bound (FUP)** per agent.

---

## 5. Agents Supported Per GPU

Using the 120 hours/month heavy‑agent FUP as the safe baseline.

### Optimised Worst Case (17,280 hrs/month)

* Heavy agents: **144**
* Normal agents: **288**
* Light agents: **864**

### Real‑World (28,800–36,000 hrs/month)

* Heavy agents: **240–300**
* Normal agents: **480–600**
* Light agents: **1,440–1,800**

### Best Case (51,840 hrs/month)

* Heavy agents: **432**
* Normal agents: **864**
* Light agents: **2,592**

---

## 6. True Internal Cost Per Agent

Based on **£550/month cost per GPU**.

### Optimised Worst Case (144 heavy agents)

**Cost per agent:** £3.82 per month

### Real‑World (240–300 heavy agents)

**Cost per agent:** £1.83–£2.29 per month

### Best Case (432 heavy agents)

**Cost per agent:** £1.27 per month

---

## 7. Per-Agent Revenue Model (Pricing Options)

Evaluating revenue at the requested price points: **£5**, **£8**, **£12** per agent.

### Optimised Worst Case (144 agents)

* £5 → Revenue £720 → Profit **£170**
* £8 → Revenue £1,152 → Profit **£602**
* £12 → Revenue £1,728 → Profit **£1,178**

### Real‑World (240 agents)

* £5 → £1,200 → Profit **£650**
* £8 → £1,920 → Profit **£1,370**
* £12 → £2,880 → Profit **£2,330**

### Real‑World (300 agents)

* £5 → £1,500 → Profit **£950**
* £8 → £2,400 → Profit **£1,850**
* £12 → £3,600 → Profit **£3,050**

### Optimised Best Case (432 agents)

* £5 → £2,160 → Profit **£1,610**
* £8 → £3,456 → Profit **£2,906**
* £12 → £5,184 → Profit **£4,634**

---

## 8. Monthly GPU Scaling Model

Linear scaling based only on GPU count.

| GPUs | Worst Case | Real-World          | Best Case   |
| ---- | ---------- | ------------------- | ----------- |
| 1    | 17,280 hrs | 28,800–36,000 hrs   | 51,840 hrs  |
| 2    | 34,560 hrs | 57,600–72,000 hrs   | 103,680 hrs |
| 5    | 86,400 hrs | 144,000–180,000 hrs | 259,200 hrs |

---

## 9. Key Takeaways

* Maximum GPU cost = **£550/month** regardless of workload.
* One GPU can process **17k–52k hours of audio per month**.
* Cost per audio-hour = **£0.03 → £0.01**.
* Cost per agent = **£3.82 → £1.27**.
* Sustainable retail price = **£12–£20 per agent per month**.
* Margin per GPU = **£600 → £4,600 per GPU per month** depending on load.

This architecture produces extremely strong margin at scale and protects Conversant’s cost base in all utilisation scenarios.

---

## 10. Use Cases

This architecture supports:

* Contact centres (medium–large)
* Sales teams
* Service desks
* Support queues
* Any high‑volume telephony environment

---

## 11. Next Steps

* Deploy SQS → GPU Autoscaling → Whisper v3 worker
* Validate concurrency behaviour on production audio
* Define FUP at 120 hrs/month per agent
* Roll out agent-based pricing across CCI Lite
* Implement GPU scaling policy thresholds for >1 GPU clusters

---

This document is now the authoritative reference for:

* GPU transcription performance
* Cost modelling
* Agent capacity
* Per-agent pricing
* Scaling strategy
