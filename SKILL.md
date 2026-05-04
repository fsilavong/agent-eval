---
name: agent-eval
description: >
  Evaluate agentic systems at component and end-to-end level, producing a
  structured report with metrics, datasets, and regression strategy. Use this
  skill whenever the user wants to evaluate, benchmark, or test any agentic or
  AI pipeline system, including RAG pipelines, multi-agent systems, LLM
  chains, retrieval systems, or any system with modelling components.
  Triggers include: "evaluate my agent", "benchmark my pipeline", "write evals
  for", "test my RAG", "how is my system performing", and "set up evals".
  Always use this skill when the user shares a codebase and asks how well it
  works.
---

# Agent Evaluation Skill

Evaluate any agentic or AI pipeline system at both **component level** and **end-to-end level**, producing a comprehensive, reproducible report.

Before starting, read all reference files:
→ `references/doc_template.md` — report structure for `doc.md`
→ `references/evalcase_schema.md` — EvalCase schema and difficulty axes
→ `references/augmentation_techniques.md` — augmentation technique catalog

---

## Communication Guide

This skill involves concepts that may be unfamiliar to users who are not ML evaluation experts. Follow these guidelines throughout:

- **Explain why before you act.** At the start of each phase, briefly say what you're about to do and why it matters. e.g. _"We're choosing metrics now so we have a clear definition of success before generating any data. This prevents building a dataset around the wrong question."_
- **Surface decisions, not just outputs.** When you make a judgment call (which metric, how many categories, what difficulty means for this system), name it explicitly and invite pushback. e.g. _"I'm proposing faithfulness as the metric for the summarizer because hallucination is the main failure mode here — does that match your concern?"_
- **Preview before investing.** Before running any expensive step (dataset generation, full eval run), show a sample and confirm the user is happy with it.
- **Translate scores into plain language.** After reporting numbers, always add a sentence about what they mean in practice. e.g. _"0.67 faithfulness means roughly 1 in 3 summaries contains a claim not supported by the source text."_
- **Flag surprises explicitly.** If a score is unexpected, say so and explain why before moving on.

---

## Folder Structure

All evaluation artefacts live under `agent-eval/` in the user's project root:

```
agent-eval/
    data/
        generate.<ext>          # reproducible dataset generation script
        cases.jsonl
        component-level/
            <component-name>/
                manifest.json   # records which end-to-end run produced these inputs
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
   - Whether each component can be invoked independently with controlled inputs (**decomposable**) or only as part of the full agent loop (**black-box**). Document this in doc.md Section 1.

2. Write your understanding into `agent-eval/doc.md` using the structure in `references/doc_template.md` — fill in sections 1 (System Introduction) and 2 (Key Components) now. Leave later sections as placeholders.

3. **Ask only what the code cannot answer.** Based on what you learned, identify the genuine gaps — things that are ambiguous in the code, depend on user intent, or require domain context. Ask only those. Frame your questions with what you already know so the user doesn't repeat themselves:

   > _"From reading the code I can see [X, Y, Z]. A few things I couldn't determine:_
   > _1. [Gap — e.g. "What does a bad response look like to you? The code doesn't define acceptable quality."]_
   > _2. [Gap — e.g. "Are there input types that matter more than others for your use case?"]_
   > _3. [Gap — e.g. "Do you have production logs, saved inputs, or historical runs I could sample from?"]"_

   Common gaps worth asking about if not clear from code: the user's definition of quality, known failure modes they've already observed, whether production data is available, and any edge cases they particularly care about. Do not ask about things the code already tells you.

---

### Phase 2 — Choose Metrics

4. For each component and for the end-to-end pipeline, select the single best evaluation metric. Choose based on:
   - The user's answers from Phase 1 step 3 (especially their definition of "bad" and known failure modes)
   - The system's goal (e.g. faithfulness for RAG, F1 for classification)
   - Industry standard for the task type

   Present your metric choices and ask for sign-off before proceeding: _"Here's what I'm proposing to measure for each component, and why. Does this match what you care about?"_

   Document metric choices with brief justification in `agent-eval/doc.md` section 2 and 3.

---

#### Step 4b — Judge Alignment (for any LLM-as-judge metric)

If LLM-as-judge is used to score, **do this step before building the dataset**. It takes a few minutes and prevents discovering a miscalibrated judge after scoring a large dataset.

Explain to the user: _"Before we generate any data, I want to show you a few examples of how the judge will score outputs. LLM judges reflect the rubric we give them — not a universal standard. This step aligns the rubric to your definition of quality."_

a. Select or construct a small set of test cases covering:
   - One that **should score well** (a clearly good output)
   - One that **should score poorly** (hallucination, wrong answer, irrelevant)
   - One that is **borderline** (plausible but imperfect)

b. Run each through the judge. Display each result clearly:

   ```
   Case 1 — expected: HIGH
   Input:         [full input]
   System output: [full output]
   Judge score:   X.XX
   Judge says:    "[quoted or paraphrased rationale]"
   ```

c. Ask: _"Does this match your intuition? Is the judge too strict, too lenient, or off on any dimension?"_

d. If the user disagrees: revise the rubric prompt, re-run, and show updated results. Repeat until aligned.

e. Only proceed to Phase 3 once the user signs off. Record the final rubric as a versioned constant in `metrics/`.

---

### Phase 3 — Build the Dataset

Objective: Build a representative evaluation dataset that turns “good” into measurable test cases, so we can fairly compare changes and quantify whether the agent improves on the inputs and failure modes that matter. 

All dataset preparation logic lives in `data/generate.<ext>`. All prompt templates referenced in the schema are stored as versioned constants inside this script. Every LLM call records its model, prompt template name, and seed. The final output is a single `data/cases.jsonl` — one EvalCase record per line. See `references/evalcase_schema.md` for the full schema definition.

Pipeline — run sequentially inside `data/generate.<ext>`:

5. Ask the user: _"Do you have production data I can pull from — e.g. saved inputs, execution logs, historical runs, or a database of past cases? If so, where does it live?"_

**If YES — sample from production:**

  a. Pull and randomly sample using a fixed seed (versioned constant in code). The pull and sampling logic lives entirely in `generate.<ext>`.

  b. Study all sampled inputs and define categories based on **input characteristics** — reasoning complexity, signal ambiguity, scope, domain — not pipeline stage names. Assign difficulty based on the axes in the EvalCase schema. Document in `agent-eval/doc.md` section 4.

  c. From each category, select **an appropriate number of representative seed cases**, balanced across difficulty (easy / medium / hard).

**Synthetic Generation:**

  a. Define **distinct input categories** grounded in input characteristics. If production inputs are available, conduct a gap analysis for missing or underrepresented categories.

  b. For each new / underrepresented category, generate **representative seed cases**, using real production inputs as style reference if available, balanced across difficulty levels.

---

#### Step 5b — Dataset Preview (gate before augmentation)

Before running the full augmentation pipeline, show a preview and get sign-off.

Explain to the user: _"Categories determine what failure modes we can detect and how we'll slice the results. It's much easier to change them now than to regenerate the full dataset later."_

a. Present the proposed categories in plain language, connecting each to what the user cares about from Phase 1:
   ```
   Proposed categories:
   - [category-a]: [description] — covers [user's concern]
   - [category-b]: [description]
   ...
   ```

b. Generate a few sample inputs (one or two per category) and show them:
   ```
   Samples:
   [category-a, easy]:   ...
   [category-a, hard]:   ...
   [category-b, medium]: ...
   ```

c. Ask: _"Do these categories cover the inputs that matter most to you? Do the samples look like real inputs your system would receive? Anything to add, remove, or rename?"_

d. Incorporate feedback. Proceed to full generation only after the user confirms.

---

6. From the seed set, generate an augmented dataset by selecting and applying techniques per seed via LLM — enough to give each category sufficient coverage for statistical tests, balanced across difficulty levels. See `references/augmentation_techniques.md` for the full technique catalog with guidance on when each applies.

   Maintain balanced difficulty distribution.

   **Dataset support table**: after `cases.jsonl` is finalised, compute per-category × difficulty counts and surface them in `agent-eval/doc.md` section 4.

   **Component-level test data**: do not construct from scratch. After the first end-to-end run (Phase 5, step 8), the intermediate output of each component is cached to disk. Use those cached outputs as input test cases for each component's isolated eval. Save to `data/component-level/<component>/` with a `manifest.json` recording the source run. If the end-to-end dataset is regenerated, check manifests — regenerate component-level data if stale.

---

### Phase 4 — Write the Evaluation Code

7. Write the following in the codebase's language, keeping code **concise and readable**:

   - **Dataset loader** — reads from `data/` directories
   - **Component-level eval** _(decomposable systems only)_ — evaluates each component in sequence, passing output of stage N as input to stage N+1. Single run per case. Skip for black-box systems.
   - **End-to-end eval** — runs the full pipeline on `data/cases.jsonl` with multiple trials per case (3 is a reasonable default; increase if variance is high). Report the mean score across trials. Store all trial outputs in the timestamped result file.
   - **Statistical testing** — after aggregating trial means, run a paired t-test (or bootstrap resampling for small samples) against the previous run's per-case scores. Report p-value and effect size. Never report a score without a 95% CI.
   - **`run.<ext>`** — single entry point that runs component eval (if applicable), end-to-end eval, stat testing and chart generation, writing all results to `runs/<YYYY-MM-DD>/result_YYYYMMDD_HHMMSS.json`. After writing, also copy (or symlink) the result to `runs/<YYYY-MM-DD>/result.json` so NEXT RUN can always find the latest result at a stable path.

   All metrics go into `metrics/` as simple, importable modules.

---

### Phase 5 — Run & Document

8. Run `run.<ext>`. Stores output in `runs/<YYYY-MM-DD>/result_YYYYMMDD_HHMMSS.json` (timestamped) and `runs/<YYYY-MM-DD>/result.json` (stable pointer to latest), including per-case trial scores, means, CIs, and git hash.

9. Analyse results:
   - Note key findings and failure modes
   - Generate charts and save to `runs/<YYYY-MM-DD>/images/`:
     - Score distribution per component (histogram)
     - Per-category score breakdown (bar chart)
     - Difficulty vs. score heatmap
     - Trial variance plot (mean ± CI per category)
   - Update `agent-eval/doc.md`:
     - **Section 5**: component-level results (with CIs) and end-to-end results — overall mean ± CI, and by category × difficulty. Cite the run file for every cell.
     - **Section 5b**: a few notable cases (highest, lowest, and optionally one surprising). For each: enough input and output to understand the failure mode, per-dimension judge scores, judge rationale, and a brief analysis. Analysis must go beyond restating the score.
     - **Section 6a**: initial trigger rules, scope, flag near-saturated categories or high trial variance
     - **Section 6b**: first row in run history (date, git hash, run file, "Initial cold start", score ± CI, p-value = —, strategy = None)
     - **Section 7 (Takeaways)**: only if findings point to a change that would make a significant difference. Each must cite the specific score or pattern from this run AND ground the recommendation in established ML evaluation knowledge — not just pattern-matching on two data points. Omit entirely if no takeaway clears this bar.
   - Output a summary table in chat (component scores, E2E overall ± CI, by-category × difficulty breakdown). Translate each score into one plain-language sentence.

---

## NEXT RUN

1. **Diff check** — use `git diff` against the git hash in the last `result.json`. Categorise changes:
   - **Trigger re-evaluation**: prompt text, model name/version, component logic, tool definitions
   - **Trigger dataset regeneration**: any prompt template in `EvalCase` schema, sampling seed, category definitions. Regenerated dataset invalidates component-level data — check manifests.
   - **No action needed**: logging, comments, UI, infrastructure
   
   List affected artefacts and confirm with the user before proceeding.

2. **Dataset review** — check whether the existing dataset still covers changed components adequately. If new categories are needed, follow COLD-START Phase 3 (Step 5b preview gate included) for affected scope only.

3. **Update eval code** — add or modify component/end-to-end eval logic. Keep `run.<ext>` as the single entry point.

4. **Run** — execute `run.<ext>`. Run paired t-test (or bootstrap resampling for small samples) against previous per-case scores. Update `doc.md`:
   - **Section 6a**: review every bullet — revise trigger rules if components changed, update scope if new failure modes emerged, flag newly saturated or high-variance categories.
   - **Section 6b**: append one row with date, git hash, run file, change description, score ± CI, p-value vs. previous run, strategy updates.
   - Output before/after comparison table in chat (overall ± CI, per category × difficulty, p-value, effect size). Flag regressions only if statistically significant at p < 0.05.

---

## Key Principles

- **Concise code only** — readable, minimal, no unnecessary abstractions
- **Sequential execution** — components run in pipeline order; intermediate outputs are passed forward and cached to disk
- **Multiple trials for end-to-end** — run each E2E case multiple times and report the mean; component evals are single-run. Increase trial count if variance is high.
- **Always report uncertainty** — every score includes a 95% CI. Use paired t-test or bootstrap resampling as appropriate for your sample size.
- **Reproducible generation** — every LLM call in `generate.<ext>` records model, prompt template, and seed. Re-running the script reproduces the full dataset.
- **Full citation** — every result in `doc.md` references its run file, dataset, and git hash.
- **Single entry point** — `run.<ext>` runs everything
- **Judge model must be stronger than the system under test** — use a model at least one capability tier above production. A peer-tier judge will systematically miscalibrate on the system's blind spots. Set `JUDGE_MODEL` as a constant; update whenever the production model is upgraded.
- **Checkpoint and resume** — `run.<ext>` writes a checkpoint after every completed case. On re-run, skip completed cases. Result files use full timestamps (`result_YYYYMMDD_HHMMSS.json`) so multiple runs on the same day never overwrite each other. After each run, `result.json` is also updated as a stable pointer to the latest timestamped file.
- **Study before asking** — read the codebase thoroughly before asking the user any questions. Ask only about genuine gaps the code cannot answer.
- **User alignment at each gate** — confirm with the user at Phase 1 (gap questions), judge alignment, and dataset preview before advancing. Each gate is cheap; re-doing downstream work is not.
- **Plain-language translation** — after every set of scores, one sentence explaining what the number means in practice.

---

## Reference Files

- `references/doc_template.md` — canonical structure for `doc.md`
- `references/evalcase_schema.md` — EvalCase schema definition with difficulty axis table
- `references/augmentation_techniques.md` — augmentation technique catalog with guidance on applicability
