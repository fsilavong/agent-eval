# [System Name] — Evaluation Report

> **Run reference**: `runs/YYYY-MM-DD/result.json`  
> **Dataset**: `data/queries.jsonl`  
> **Last updated**: YYYY-MM-DD

---

## 1. System Introduction

_A concise description of what this agentic system does, its intended users, and the problem it solves. 2–4 sentences._

**System goal**: ...  
**Input**: ...  
**Output**: ...

---

## 2. Key Components

_List each modelling component identified during codebase study. For each, document its role, input, output, and chosen evaluation metric._

| Component | Role | Input | Output | Metric |
|-----------|------|-------|--------|--------|
| `component-a` | ... | ... | ... | ... |
| `component-b` | ... | ... | ... | ... |

---

## 3. End-to-End Pipeline

_Describe the full pipeline from raw user query to final output. Include a brief flow diagram in text or Mermaid if helpful._

```
User Query → [Component A] → [Component B] → ... → Final Output
```

**End-to-end metric**: _Name and justification. Reference industry standard where applicable._

---

## 4. Dataset

### Query Categories

_Multi-label — a query can belong to multiple categories, so totals may exceed unique query count._

| Category | Description |
|----------|-------------|
| `category-a` | ... |
| `category-b` | ... |

### Dataset Support (category × difficulty)

_Number of queries per category × difficulty level (seeds + augmentations combined). Use to confirm sufficient representation for statistical tests._

| Category | Easy | Medium | Hard | Total |
|----------|------|--------|------|-------|
| `category-a` | ... | ... | ... | ... |
| `category-b` | ... | ... | ... | ... |
| **Total**    | ... | ... | ... | ... |

**Source split**: X production / Y synthetic (Z% / W%)

### Generation Configuration

_Versioned record of every input that could change between runs. Diff this section across runs to see what changed._

- **Source**: production (<where>) + synthetic supplementation
- **Sampling seed**: ...
- **Categorisation**: prompt=`<template_name>`, model=`<model>`
- **Difficulty scoring**: prompt=`<template_name>`, model=`<model>`
- **Augmentation — technique selection**: prompt=`<template_name>`, model=`<model>`
- **Augmentation — generation**: per-technique prompts (`<obscuration_v1>`, `<entity_swap_v1>`, ...)
- **Generated**: YYYY-MM-DD via `data/generate.<ext>`
- **Component-level data source**: `runs/YYYY-MM-DD/result.json` (per `data/component-level/<component>/manifest.json`)

### Pipeline Overview

_Brief context for readers; full behaviour is defined in the skill, not here._

Dataset built by pulling production queries, randomly sampling at a fixed seed, applying LLM categorisation and difficulty scoring, identifying gaps via category × difficulty distribution, supplementing with synthetic generation, then augmenting each seed via 3–4 LLM-selected techniques. Difficulty re-scored on augmented queries. Full per-record lineage in `data/queries.jsonl`.

---

## 5. Test Results

_Cite the exact run file and dataset used for every result reported._

### Component-Level

| Component | Metric | Score | 95% CI | Run |
|-----------|--------|-------|--------|-----|
| `component-a` | ... | ... | ... | `runs/YYYY-MM-DD/result.json` |
| `component-b` | ... | ... | ... | `runs/YYYY-MM-DD/result.json` |

### End-to-End

_3 trials per query. Score = mean across trials. All scores reported with 95% CI._

**Overall**

| Metric | Mean Score | 95% CI | n queries | Run |
|--------|------------|--------|-----------|-----|
| ... | ... | ... | ... | `runs/YYYY-MM-DD/result.json` |

**By category × difficulty**

| Category | Easy (mean ± CI) | Medium (mean ± CI) | Hard (mean ± CI) | Overall |
|----------|------------------|--------------------|--------------------|---------|
| `category-a` | ... | ... | ... | ... |
| `category-b` | ... | ... | ... | ... |
| **Overall**  | ... | ... | ... | ... |

_Charts: see `runs/YYYY-MM-DD/images/` — score distribution, per-category breakdown, difficulty heatmap, trial variance._

### Observations

_Key findings, failure modes, surprising results. Be specific._

---

## 6a. Regression Test Strategy

_The living rules for how future runs should be conducted. Review and update this section after every run — revise rules when the system changes, categories saturate, or variance patterns shift._

- **Re-evaluation triggers**: prompt text, model name/version, component logic, tool definitions. Update this list if new components are added or scope changes.
- **Dataset regeneration triggers**: any prompt template referenced in the `UserQuery` schema (categorisation, difficulty scoring, augmentation technique selection, augmentation generation), the sampling seed, or category definitions. A regenerated dataset invalidates component-level data — check manifests.
- **Scope**: Which components and categories to re-evaluate for each type of change. Refine after each run based on what proved sensitive vs. stable.
- **Pass criteria**: A drop is a regression if statistically significant at p < 0.05 (paired t-test if n ≥ 30, bootstrap with 1,000 resamples if n < 30). Always report effect size — a significant but tiny drop may not warrant action.
- **Saturation watch**: If any category reaches a near-perfect score (>95%), flag it to graduate from capability eval to regression suite — it no longer measures headroom, only backsliding.
- **Variance watch**: If trial variance is high for a category (wide CI), note it here and consider whether 3 trials is sufficient or the dataset needs expanding for that category.

_Update instructions: after each run, review each bullet above. Revise trigger rules if components changed, update scope if new failure modes were found, flag any saturated or high-variance categories. Record what changed and why in 6b._

---

## 6b. Run History

_Append-only. Add one row after every run. Never edit past rows._

| Date | Git Hash | Run File | Change | E2E Score (mean ± 95% CI) | p-value vs. prev | Strategy updates | Notes |
|------|----------|----------|--------|---------------------------|------------------|------------------|-------|
| YYYY-MM-DD | `abc1234` | `runs/YYYY-MM-DD/result.json` | Initial cold start | ... | — | None | — |