---
layout: post
title: "tabular_embedding_01 — learned embeddings for structured data"
---

Short note on `tabular_embedding_01`, a 2025 project (May–Oct, private). ~107 R-B commits. The repo does not have a public README; this is the shape from the inside.

## The problem

A lot of financial data is tabular — features are a mix of numeric, categorical, and ordered types, often with missing values and column meanings that shift over time. Tree-based models handle that gracefully; neural models don't, without work. The question was whether a learned embedding layer per column, trained jointly with a downstream head, could close the gap — and in particular whether a pretrained tabular transformer (TabPFN-shaped) could serve as a base that you fine-tune on your specific dataset.

## What I built

- Per-column embedding modules sized to the column's type and cardinality, with a shared downstream head.
- A fine-tune loop for a TabPFN-shaped backbone against a finance-flavoured downstream task.
- Baselines: XGBoost, a plain MLP with one-hot encoding, and a version with the learned embeddings but no pretrained backbone. Three numbers next to each other on each run.

## Where it plateaued

- On the finance-flavoured tasks I threw at it, XGBoost stayed the cheapest honest baseline. The learned embeddings closed some of the gap; the pretrained backbone sometimes helped, sometimes didn't, depending on how close the downstream distribution was to the pretraining distribution.
- The embedding layer is a real feature where you want to combine tabular data with text or with a time series — the joint model has a place to put the tabular side without flattening it. On pure tabular tasks, it's often net negative versus a good tree.

## What I'd claim

- A working training pipeline, with baselines, on several datasets. Not production; not a research result worth writing up on its own.
- The project was useful as a forcing function for thinking about the finance_lab input shapes later — specifically, what a "feature column" is once you've committed to neural models.

## What I wouldn't claim

- State-of-the-art on a benchmark. I didn't run Open Tabular Benchmark; the comparisons were against internal baselines.
- Pretrained-backbone is a silver bullet. For my datasets, it was one tool among several, not a clear winner.

Private repo.
