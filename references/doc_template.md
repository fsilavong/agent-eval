# [System Name] — Evaluation Report

> **Run reference**: `runs/YYYY-MM-DD/result.json`  
> **Dataset**: `data/end-to-end/` · `data/component-level/`  
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

_Each category should be meaningfully distinct from the others._

| Category | Description | # Queries |
|----------|-------------|-----------|
| `category-a` | ... | 10 |
| `category-b` | ... | 10 |

### Generation Methodology

- **Source**: _(Synthetic / sampled from production / mixed)_
- **Seed set**: 10 queries per category, balanced across easy / medium / hard (min 1–2 per level)
- **Difficulty rating**: each query scored across three axes — query clarity (explicit → ambiguous), reasoning depth (single-step → multi-hop), domain specificity (common knowledge → expert). Easy = low on all three; hard = high on two or more; medium = otherwise.
- **Augmentation**: 3–4 techniques applied per seed (obscuration, entity swapping, format/style variation, negation injection, compositional queries, noise injection, specificity variation), targeting ~50 queries per category. Technique selection guided by query type.
- **Component-level data**: derived from cached intermediate outputs of the first end-to-end run, not constructed independently.

### Dataset Files

| File | Description |
|------|-------------|
| `data/end-to-end/` | Full pipeline test queries |
| `data/component-level/<component>/` | Isolated component test cases |

---

## 5. Test Results

_Cite the exact run file and dataset used for every result reported._

### Component-Level

| Component | Metric | Score | Run |
|-----------|--------|-------|-----|
| `component-a` | ... | ... | `runs/YYYY-MM-DD/result.json` |
| `component-b` | ... | ... | `runs/YYYY-MM-DD/result.json` |

### End-to-End

_3 trials per query. Score = mean across trials. All scores reported with 95% CI._

| Category | Metric | Mean Score | 95% CI | Trials | Run |
|----------|--------|------------|--------|--------|-----|
| `category-a` | ... | ... | ... | 3 | `runs/YYYY-MM-DD/result.json` |
| `category-b` | ... | ... | ... | 3 | `runs/YYYY-MM-DD/result.json` |

_Charts: see `runs/YYYY-MM-DD/images/` — score distribution, per-category breakdown, difficulty heatmap, trial variance._

### Observations

_Key findings, failure modes, surprising results. Be specific._

---

## 6a. Regression Test Strategy

_The living rules for how future runs should be conducted. Review and update this section after every run — revise rules when the system changes, categories saturate, or variance patterns shift._

- **Trigger**: What changes warrant a re-run. Default: prompt text, model name/version, component logic, tool definitions. Update this list if new components are added or scope changes.
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
