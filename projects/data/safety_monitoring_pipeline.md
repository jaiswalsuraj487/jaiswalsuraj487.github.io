---
title: "Real-Time Safety Monitoring Pipeline"
author: "Suraj Jaiswal"
date: "April 1, 2025"
format: html
categories: ['Computer Vision', 'MLOps']
---

An edge-deployed, real-time compliance monitoring system for a chemical factory where workers operate on hazardous chemical containers. The system enforces multiple safety use cases simultaneously using a modular deep learning pipeline.

---

## Use Cases

- **PPE compliance:** Detect whether workers wear safety vest and hard hat
- **Harness & lanyard:** Verify workers on containers are tethered to the ceiling via harness and lanyard
- **Container tracking:** Detect ingoing/outgoing trucks with containers and extract ISO numbers via OCR

---

## Method

| Component | Role |
|---|---|
| **YOLOX** | Person and container detection |
| **MobileNet** | Safety vest, hard hat, harness classification |
| **OCSort** | Stable multi-object tracking for unique ID assignment |
| **Segmentation** | Inclusion/exclusion zone identification |
| **OCR** | Parse ISO numbers from container surfaces |
| **Qwen 7B (quantized VLM)** | Lanyard detection (thin rope — classical models insufficient) |

---

## Implementation

**Dataset Collection:**
- Recorded video from live CCTV RTSP streams
- Ran GroundingDINO auto-annotation over videos to extract person and container crops
- Parent person crop used to generate safety vest, hard hat, and harness instances
- Validated and cleaned dataset before training

**Inference Flow — PPE:**
```
Video → extract frame → YOLOX (person) → OCSort (tracking)
→ parallel: (a) safety_vest  (b) hard_hat  (c) harness
→ if harness detected → Qwen 7B VLM for lanyard check
```

**Inference Flow — Container ISO:**
```
Video → extract frames → YOLOX (container) → OCSort (tracking)
→ OCR on container crop → extract ISO number
```

> Qwen 7B VLM is called **only when harness is detected** to minimize API calls (each VLM call takes 2–3 seconds).

---

## Deployment Architecture

Hosted on **Triton Server** with separate Docker containers per module on an edge device:

| Container | Function |
|---|---|
| `s3_sync` | Syncs model versions and run configs with AWS S3 |
| `triton_hosting` | Queue-based async, scalable model request processing |
| `vlm_inference` | Hosts Qwen7B quantized (called conditionally) |
| `use_case_aggregation` | Runs main inference loop, saves JSON annotation logs |
| `violation_detection` | Checks intervals — flags violations if PPE absent > N seconds, pushes to frontend DB |

---

## Outcomes

- Detected safety vest, hard hat, harness, and lanyard compliance in real time
- Container ISO numbers accurately extracted via OCR
- Interval-based violation detection reduced false positives
- Modular pipeline handled multiple workers and containers simultaneously with stable tracking

**Tech Stack:** `Python` `YOLOX` `MobileNet` `OCSort` `GroundingDINO` `Qwen 7B` `Triton Server` `Docker` `AWS S3` `AWS IoT Core` `OCR`

---

## Conclusions

A modular, edge-deployed pipeline combining detection (YOLOX), tracking (OCSort), classification (MobileNet), and VLM-based reasoning (Qwen7B) is highly effective for real-time multi-use-case industrial compliance. Conditional VLM inference significantly reduced latency and API costs without compromising accuracy.

**Known limitations:** Lanyard detection is challenging due to its thin, flexible nature. VLM inference latency (2–3s/call) is a bottleneck. Performance degrades in poor lighting or suboptimal CCTV placement.
