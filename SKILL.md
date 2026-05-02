---
name: agent-eval
description: Evaluate agentic systems at component and end-to-end level, producing a structured report with metrics, datasets, and regression strategy. Use this skill whenever the user wants to evaluate, benchmark, or test any agentic or AI pipeline system — including RAG pipelines, multi-agent systems, LLM chains, retrieval systems, or any system with modelling components. Triggers include: "evaluate my agent", "benchmark my pipeline", "write evals for", "test my RAG", "how is my system performing", "set up evals", or any request to measure or compare AI system performance. Always use this skill when the user shares a codebase and asks how well it works.
---

# Agent Evaluation Skill

Evaluate any agentic or AI pipeline system at both **component level** and **end-to-end level**, producing a comprehensive, reproducible report.

Before starting, read the doc template for report structure:
→ `references/doc_template.md`

---

## Folder Structure

All evaluation artefacts live under `agent-eval/` in the user's project root:

```
agent-eval/
    data/
        generate.<ext>          # reproducible dataset generation script
        queries.jsonl           
        end-to-end/
        component-level/
            <component-name>/
    metrics/
    runs/
        <YYYY-MM-DD>/
            result.json
            images/          # optional charts
    run.<ext>                # entry point in the codebase's language
    doc.md
```

---

## Mode Detection

**Before doing anything**, check whether `agent-eval/` exists in the codebase.

- **Does not exist** → follow [COLD-START](#cold-start)
- **Exists** → confirm with the user: _"I can see a previous eval run exists. Do you want to continue from where you left off (NEXT RUN), or start fresh (COLD-START)?"_  
  Proceed based on their answer.

---

## COLD-START

### Phase 1 — Understand the System

1. Study the codebase thoroughly. Identify:
   - The system's overall intent and goal
   - Every key modelling component (e.g. retriever, reranker, generator, classifier)
   - The input and output of each component
   - The primary language of the codebase — all evaluation code must be written in this language

2. If the system intent is ambiguous, confirm with the user before proceeding.

3. Write your understanding into `agent-eval/doc.md` using the structure in `references/doc_template.md` — fill in sections 1 (System Introduction) and 2 (Key Components) now. Leave later sections as placeholders.

---

### Phase 2 — Choose Metrics

4. For each component and for the end-to-end pipeline, select the single best evaluation metric. Choose based on:
   - The system's goal (e.g. faithfulness for RAG, F1 for classification)
   - Industry standard for the task type

   Document metric choices with brief justification in `agent-eval/doc.md` section 2 and 3.

---

### Phase 3 — Build the Dataset

All dataset generation logic lives in data/generate.<ext>. Every LLM call records its model, prompt template, and seed. The final output is a single data/queries.jsonl — one UserQuery record per line.

`UserQuery` schema:
{
  id:               string,
  query:            string,
  is_real:          bool,                       # true = production, false = synthetic
  categories:       List[string],               # multi-label, e.g. ["factual", "multi-hop"]
  difficulty: {
    label:          "easy" | "medium" | "hard",
    axes: {
      clarity:      1-3,                        # 1=explicit, 3=ambiguous
      reasoning:    1-3,                        # 1=single-step, 3=multi-hop
      domain:       1-3                         # 1=common knowledge, 3=expert
    },
    rationale:          string,
    model:              string,
    prompt_template:    string                  # versioned template name
  },
  augmented_queries: [{
    query:                         string,
    technique:                     string,
    technique_selection_rationale: string,
    model:                         string,
    prompt_template:               string,      # versioned template name
    generation_seed:               int
  }],
  metadata: Any                                 # opaque field for any additional info (e.g. production query ID, augmentation lineage, etc.)
}

Pipeline — run sequentially inside data/generate.<ext>:

5. Ask the user: _"Do you have real production queries I can use as a seed set?"_

**If YES — sample from production:**

  a. Pull and randomly sample using a fixed seed (versioned constant in code) to ensure reproducibility. Sample at least 100 queries if possible. The pull and sampling logic lives entirely in generate.<ext>.

  b. Study all the sampled queries and come up with categorisation given the system intent. Classify each query into one or more categories (e.g. "factual", "multi-hop", "ambiguous", "domain-specific"), and difficulty level (easy, medium, hard) based on the axes defined in the   `UserQuery`schema. Document category definitions and difficulty label rationale in `agent-eval/doc.md` section 4.

  c. From each category, select **an appropriate number of representative seed queries**, balanced across easy / medium / hard, to use as the basis for augmentation. 

**Synthetic Generation:**


  a. Based on the system intent, define a set of **distinct query categories** that users might ask. If production queries are available, conduct a gap analysis to identify missing categories and underrepresented categories. Document category definitions in `agent-eval/doc.md` section 4.

  b. For each new / underrepresented category, generate **an appropriate number of representative seed queries** with production queries as style transfer if available, balanced across easy / medium / hard, to use as the basis for augmentation. 

6. From the seed set, generate an augmented dataset by selecting and applying **3–4 techniques per seed** via LLM, targeting ~50 queries per category total. Techniques:
   - **Obscuration** — paraphrase or use implicit phrasing to express the same intent
   - **Entity swapping** — change names, dates, domains while preserving query structure
   - **Format/style variation** — verbose, terse, question vs. instruction, formal vs. casual
   - **Negation injection** — flip intent ("what does X *not* do") to test polarity robustness
   - **Compositional queries** — combine two categories into one query to test multi-hop reasoning
   - **Noise injection** — introduce typos, grammatical errors, or informal phrasing
   - **Specificity variation** — restate the same query at a much broader and much narrower scope

   Maintain balanced difficulty distribution across the augmented set. Save end-to-end queries to `data/end-to-end/`.

   **Component-level test data**: do not construct these from scratch. After the first end-to-end run (Phase 5, step 8), the intermediate output of each component is cached to disk. Use those cached outputs as the input test cases for each component's isolated eval — this reflects the real upstream distribution each component sees in production. Save to `data/component-level/<component>/`.

   Document dataset structure, generation methodology, and difficulty label rationale in `agent-eval/doc.md` section 4.

---

### Phase 4 — Write the Evaluation Code

7. Write the following in the codebase's language, keeping code **concise and readable** — do not over-engineer:

   - **Dataset loader** — reads from `data/` directories
   - **Component-level eval** — evaluates each component in sequence, passing the output of stage N as input to stage N+1 (to reflect realistic pipeline behaviour). Single run per query.
   - **End-to-end eval** — runs the full pipeline on `data/end-to-end/` with **3 trials per query**. Report the mean score across trials as the final score for each query. This accounts for LLM non-determinism. Store all 3 trial outputs per query in `result.json`.
   - **Statistical testing** — after aggregating trial means, run a paired t-test (or bootstrap with 1,000 resamples if n < 30) against the previous run's per-query scores. Report p-value and effect size. Never report a score without a 95% confidence interval.
   - **`run.<ext>`** — single entry point that runs component eval, end-to-end eval (3 trials), stat testing, and chart generation sequentially, writing all results to `runs/<YYYY-MM-DD>/result.json`. Store the git hash (if available) in the result file.

   All metrics go into `metrics/` as simple, importable modules in the codebase's language.

---

### Phase 5 — Run & Document

8. Run `run.<ext>`. Store output in `runs/<YYYY-MM-DD>/result.json`, including per-query trial scores, means, CIs, and git hash.

9. Analyse results:
   - Note key findings and failure modes
   - Generate the following charts and save to `runs/<YYYY-MM-DD>/images/`:
     - Score distribution per component (histogram)
     - Per-category score breakdown (bar chart)
     - Difficulty vs. score heatmap (easy / medium / hard × categories)
     - Trial variance plot for end-to-end (mean ± CI per category)
   - Update `agent-eval/doc.md` as follows:
     - **Section 5**: fill in component and end-to-end results tables with scores, CIs, and run reference
     - **Section 6a**: fill in the initial trigger rules, scope, and flag any categories that are already near-saturated or showing high trial variance
     - **Section 6b**: append the first row to the run history table (date, git hash, run file, "Initial cold start", score ± CI, p-value = —, strategy updates = None)
   - Output a summary table in chat showing component scores, end-to-end scores (mean ± 95% CI), and per-category breakdown

---

## NEXT RUN

1. **Diff check** — use `git diff` against the git hash stored in the last `result.json`. If git is unavailable, fall back to file modification timestamps on component source files. Changes that **trigger re-evaluation**: prompt text, model name/version, component logic (source files in the component's module), tool definitions. Changes that **do not trigger re-evaluation**: logging, comments, UI, infrastructure. List the affected components explicitly and confirm with the user before proceeding.

2. **Dataset review** — check whether the existing dataset still covers the changed components adequately. If new categories or augmentation are needed, follow COLD-START Phase 3 for only the affected scope.

3. **Update eval code** — add or modify component/end-to-end eval logic to reflect the changes. Keep `run.<ext>` as the single entry point.

4. **Run** — execute `run.<ext>` (3 trials per end-to-end query, single run for components). Pull per-query scores from `runs/<last-date>/result.json` and run a paired t-test (or bootstrap if n < 30) against the new scores. Then update `doc.md`:
   - **Section 6a**: review every bullet — revise trigger rules if new components were added, update scope if new failure modes emerged, flag any category that newly saturated (>95%) or showed high variance. Record a one-line summary of what changed and why.
   - **Section 6b**: append one row with date, git hash, run file, description of what changed, score ± CI, p-value vs. previous run, and a brief note on any strategy updates made to 6a.
   - Output a before/after comparison table in chat with mean ± CI per component and category, p-value, and effect size. Generate a before/after delta chart saved to `runs/<YYYY-MM-DD>/images/`. Flag a regression only if the drop is statistically significant at p < 0.05 — report effect size alongside so small-but-significant drops can be weighed in context.

---

## Key Principles

- **Concise code only** — readable, minimal, no unnecessary abstractions
- **Sequential execution** — components run in pipeline order; intermediate outputs are passed forward and cached to disk
- **3 trials for end-to-end** — run each end-to-end query 3 times; report mean score per query. Component evals are single-run.
- **Always report uncertainty** — every score must include a 95% CI. Use paired t-test (n ≥ 30) or bootstrap with 1,000 resamples (n < 30) for regression comparisons.
- **Reproducible generation** — every LLM call in generate.<ext> records model, prompt template, and seed. Sampling uses a fixed seed. Re-running the script reproduces the full dataset.
- **Full citation** — every result in `doc.md` must reference its run file, dataset, and git hash (if available)
- **Single entry point** — `run.<ext>` runs everything; no scattered scripts

---

## Reference Files

- `references/doc_template.md` — canonical structure for `doc.md`; read this at the start of every session