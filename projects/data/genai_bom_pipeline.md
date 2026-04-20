---
title: "GenAI Bill-of-Materials Automation Pipeline"
author: "Suraj Jaiswal"
date: "February 1, 2025"
format: html
categories: ['GenAI', 'MLOps']
---

Designed and deployed a GenAI pipeline at [Tiger Analytics](https://www.tigeranalytics.com/) to automate Bill-of-Materials (BoM) generation for lighting parts — replacing a largely manual, time-intensive process.

**Problem:** Generating structured BoMs for lighting components required engineers to manually extract and organize part specifications from unstructured documents, taking significant time per product.

**Solution:**

- LLM-based structured extraction pipeline using **AWS Bedrock** to parse product datasheets and specifications
- Experiment tracking and model versioning with **MLflow**
- Containerised with **Docker** for reproducible, production-ready deployment
- Output structured BoMs with part names, specifications, quantities, and supplier info

**Key Results:**

- **80% reduction** in manual BoM generation time
- Consistent structured output across diverse document formats (PDFs, HTML specs, Excel sheets)

**Tech Stack:** `Python` `AWS Bedrock` `MLflow` `Docker` `Prompt Engineering` `Structured Extraction`

**Impact:** Freed up engineering time from repetitive data entry, enabling faster product configuration and procurement workflows.
