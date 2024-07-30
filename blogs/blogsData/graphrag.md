---
title: "Graph RAG"
author: "Suraj Jaiswal"
date: "July 30, 2024"
format: html
categories: ['NLP']
---
# From Local to Global: A Graph RAG Approach to Query-Focused Summarization
Approach uses an LLM to build a graph-based text index in two stages: first to derive an entity knowledge graph from the source documents, then to pregenerate community summaries for all groups of closely-related entities. Given a question, each community summary is used to generate a partial response, before all partial responses are again summarized in a final response to the user.
Blog: https://www.microsoft.com/en-us/research/publication/from-local-to-global-a-graph-rag-approach-to-query-focused-summarization/

Paper link: https://arxiv.org/abs/2404.16130

Git repo: https://github.com/microsoft/graphrag

## For quick start:
We will create and use use graphrag environment.
```bash
conda create -n graphrag python=3.11
# To activate this environment, use
conda activate graphrag
```
To install the required packages, run the following command:
```bash
pip install graphrag rich
```

Add openai key value in variable GRAPHRAG_API_KEY in .env file

configure settings.yaml file  

change tokens_per_minute in setting.yaml file as per below 
https://platform.openai.com/docs/guides/rate-limits/usage-tiers


# To initialize the project in the current folder:
```bash
python -m graphrag.index --init --root .
```
You should have you input files in ./input folder in current directory
# To start graphrag indexing. using below command the data in ./input folder will be indexed
```bash
python -m graphrag.index --root .
```
To try global search using command:
```bash
python -m graphrag.query --root . --method global "what are the top methods in this study?"
```
Also Check:  
https://github.com/microsoft/graphrag/blob/main/examples_notebooks/global_search.ipynb

To try local search using command:
```bash
python -m graphrag.query --root . --method local "what are the top methods in this study?"
```
Also Check: 
https://github.com/microsoft/graphrag/blob/main/examples_notebooks/local_search.ipynb