---
title: "Medical RAG System for Automated Billing Code Generation"
author: "Suraj Jaiswal"
date: "August 1, 2024"
format: html
categories: ['NLP', 'GenAI', 'Healthcare']
---

Built a Retrieval-Augmented Generation (RAG) system at [NeuroReef Labs](https://www.linkedin.com/company/neuroreef-labs/) to automate medical billing code generation from clinical notes and patient visit records.

**Problem:** Medical coders manually assign ICD-10, CPT, SNOMED, and HCC codes to every patient visit — a slow, error-prone process that delays billing and reimbursement.

**Solution:**

- RAG pipeline over clinical documentation to retrieve relevant coding guidelines and past examples
- LLM generates billing codes grounded in retrieved context, reducing hallucinations
- **HyDE (Hypothetical Document Embeddings)** used in the medical chatbot over Athena EHR data for improved retrieval quality
- Applied **few-shot prompt tuning** for medical guideline-consistent outputs

**Codes Automated:**

| Code Type | Purpose |
|---|---|
| ICD-10 | Diagnosis classification |
| CPT | Procedure billing |
| SNOMED | Clinical terminology |
| HCC | Risk adjustment for Medicare |

**Tech Stack:** `Python` `OpenAI API` `RAG` `HyDE` `HuggingFace` `Athena EHR` `Prompt Engineering`

**Impact:** Reduced manual coding effort for medical billing, improving accuracy and turnaround time for reimbursement workflows.
