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
  Always use this skill when the user shares a codebase and asks how well it works.
---

# Agent Evaluation Skill

Evaluate any agentic or AI pipeline system at both **component level** and **end-to-end level**, producing a comprehensive, reproducible report.

Read now (needed throughout):
→ `references/doc_template.md`
→ `references/evalcase_schema.md`

---

## Communication Guide

- **Explain why before you act.** At the start of each phase, briefly say what you're about to do and why it matters.
- **Surface decisions, not just outputs.** Name judgment calls explicitly and invite pushback.
- **Preview before investing.** Before any expensive step, show a sample and confirm.
- **Translate scores into plain language.** After reporting numbers, add a sentence on what they mean in practice.
- **Flag surprises explicitly.** If a score is unexpected, say so before moving on.
- **Every gate has a skip option.** Never hold the user at a step. If they want to move on, proceed.

---

## Folder Structure

```
agent-eval/
    data/
        generate.<ext>
        cases.jsonl
        component-level/<component-name>/manifest.json
    metrics/
    runs/<YYYY-MM-DD>/
        result_YYYYMMDD_HHMMSS.json    # one per run; immutable
        result.json                    # copy of the most recent timestamped file
        annotations.jsonl              # appended by the transcript tool
        images/
    run.<ext>
    doc.md
```

---

## Mode Detection

Check whether `agent-eval/` exists in the codebase.

- **Does not exist** → ask:
  > _"Want to jump straight in (Quick Start — I'll generate some starter cases and run them so you can see what the agent does), or walk through the full setup together?"_
  - **Quick Start** → [QUICK START](#quick-start)
  - **Full setup** → [COLD-START](#cold-start)

- **Exists** → confirm: _"I can see a previous eval run. Continue from where you left off (NEXT RUN), or start fresh (COLD-START)?"_

---

## QUICK START

For users who want to see results before committing to a full setup.

1. Study the codebase. Identify the agent's purpose, inputs, outputs, and key tools.
2. Infer obvious input categories. Write the initial `data/generate.<ext>` — a starter script that generates 10–15 diverse cases and writes them to `data/cases.jsonl`. Every LLM call must record its model, prompt template, and seed. This is the beginning of the generation script; Phase 3 will expand it with the full pipeline if the user continues to full setup.
3. Write minimal eval code and run it immediately.
4. Open the transcript tool (see `references/transcript_tool.md`) showing the results. The annotation gate **must** happen through the transcript tool — not as a chat Q&A. Do not skip the tool.
5. The user annotates in freeform via the tool. Parse annotations into assertions and test case seeds.
6. Offer to continue into COLD-START (full setup) or stop here.

---

## COLD-START

### Phase 1 — Understand the System

1. Study the codebase thoroughly. Identify:
   - The system's overall intent and goal
   - Every key modelling component and its input/output
   - The primary language — all eval code must be in this language
   - Whether each component is **decomposable** or **black-box**. Document in `doc.md` section 1.
   - **State isolation**: does the agent write to external systems (DB, APIs, files)? Note what needs resetting between trials. Scaffold setup/teardown in `run.<ext>`. Only ask the user if the isolation strategy cannot be determined from the code.

2. Write `agent-eval/doc.md` sections 1 and 2. Leave later sections as placeholders.

3. **Ask only what the code cannot answer.** Frame with what you already know:
   > _"From reading the code I can see [X, Y, Z]. A few genuine gaps:_
   > _1. [e.g. What does a bad response look like to you?]_
   > _2. [e.g. Are there input types that matter more than others?]_
   > _3. [e.g. Do you have production logs I could sample from?]"_

---

### Phase 2 — Choose Metrics and Draft Rubrics

Read now: → `references/graders.md`

4. For each component and the end-to-end pipeline, select a **primary metric** — the single number used for regression decisions and comparing runs. Where a component has genuinely orthogonal dimensions (e.g. task completion and tone for a conversational agent), add **diagnostic metrics** that explain *why* the primary moved, but don't gate decisions on them. Present choices and ask for sign-off before proceeding. Document in `doc.md` sections 2 and 3.

   Avoid metric sprawl: if you're measuring more than 3–4 things per component, you're likely measuring proxies rather than what actually matters.

5. **Draft suite-level rubrics** based on your understanding of the system — generic quality dimensions that apply to every case. Show to the user:
   > _"Here are the quality dimensions I'd grade every response on. Do these match what matters to you?"_

   Calibrate rubrics before scaling: test each on a good, a bad, and a borderline case. Adjust until the user agrees with the scores.

---

### Phase 3 — Build the Dataset

Objective: build a representative eval dataset. All logic in `data/generate.<ext>`. Output: `data/cases.jsonl`.

6. Ask: _"Do you have production data — saved inputs, logs, historical runs? If so, where?"_

   **If YES** — sample (random, fixed seed). Study inputs, define categories from input characteristics, select seed cases balanced across difficulty.

   **If NO** — interview before generating:
   > _"What manual checks do you run before shipping?"_
   > _"What went wrong recently that you had to fix?"_
   > _"What would make you say 'this is broken'?"_

   Each answer is a test case. Most users have 10–20 in their heads already. Capture those first, then fill gaps synthetically.

7. **For each case, draft assertions** — specific criteria this case's response must meet — based on what you know the task requires. Show to user for refinement. See `references/evalcase_schema.md`.

8. **Write the seed section of `data/generate.<ext>`** — a script function that generates the seed cases with a fixed seed, and records model and prompt template for every LLM call. Run it to produce the initial `data/cases.jsonl`. This is the beginning of the generation script; the augmentation step below will expand it.

#### Dataset Preview (gate before augmentation)

Show proposed categories, 1–2 sample inputs per category, and ask for sign-off. Incorporate feedback before full generation.

Read now: → `references/augmentation_techniques.md`

9. Expand `data/generate.<ext>` with augmentation logic. Augment seed cases using LLM-selected techniques. Maintain balanced difficulty. Target ~50 cases per category. Log technique selection rationale and prompt template per variant. If skipping any techniques, surface the reason to the user at the preview gate — do not decide unilaterally.

   **Dataset support table**: compute per-category × difficulty counts. Surface in `doc.md` section 4.

   **Component-level test data**: after the first E2E run (Phase 5), cache intermediate component outputs to `data/component-level/<component>/` with a `manifest.json`. Use as inputs for isolated component evals. Regenerate if dataset changes.

---

### Phase 4 — Write the Evaluation Code

9. Write in the codebase's language:

   - **Dataset loader** — reads from `data/`
   - **Component-level eval** _(decomposable only)_ — single run per case, pipeline order, outputs passed forward
   - **End-to-end eval** — multiple trials per case (default 3; increase if variance is high). Report mean. Each trial must persist the **full conversation** — user input, every assistant turn (including text between tool calls), tool calls with arguments, tool outputs, and the final reply — in a **standard message format** (e.g. OpenAI Chat Completions, OpenAI Responses items, or Anthropic Messages); record which format. Carry `case.input` through to the trial record. Storing only the final response loses the information needed to debug failures and grade tool use.
   - **Graders** — per `references/graders.md`: check per-case assertions (one LLM call per assertion), score suite-level rubrics (one LLM call per dimension per case), run code-based checks. Never bundle multiple grading dimensions into one LLM call.
   - **Statistical testing** — paired t-test (n ≥ 30) or bootstrap resampling (n < 30) vs previous run. Report p-value and effect size. Never report a score without 95% CI.
   - **Concurrency** — eval runs are I/O-bound on LLM calls and become impractical past ~20 cases if run serially. Parallelise across cases (independent) and across grader calls within a trial (assertions and rubric dimensions are independent). Use a thread pool with a configurable `--workers` flag (default 8). Handle provider rate limits with exponential backoff on 429s. Serialise the checkpoint write under a lock. Trials of the same case can also run in parallel if rate limits allow. Concurrency cuts wall time, not API spend — for grading at scale, consider the provider's batch API.
   - **`run.<ext>`** — single entry point. Writes an immutable `runs/<YYYY-MM-DD>/result_<timestamp>.json` per run and copies it to `runs/<YYYY-MM-DD>/result.json` as a stable pointer to the latest run; the transcript tool, charts, and paired comparisons read from `result.json`. Checkpoints after every completed case; re-run skips completed cases.

---

### Phase 5 — Run & Document

Read now: → `references/transcript_tool.md`

10. Run `run.<ext>`.

    **0% pass rate**: if any task fails across all trials, flag before reporting aggregates:
    > _"[task-id] failed every trial. This almost always means the task spec or grader is wrong, not the agent. Open the transcript tool to inspect before drawing conclusions."_
    Exclude flagged tasks from aggregate scores until reviewed.

11. Analyse results:
    - Note key findings and failure modes
    - Generate charts to `runs/<YYYY-MM-DD>/images/`: score distribution, per-category breakdown, difficulty heatmap, trial variance
    - Update `doc.md` sections 5, 5b, 6a, 6b, 7
    - Output summary table in chat. Translate each score into one plain-language sentence.

---

## NEXT RUN

1. **Diff check** — compare vs git hash in last `result.json`. Categorise:
   - **Re-evaluate**: prompt text, model, component logic, tool definitions
   - **Regenerate dataset**: prompt templates in EvalCase schema, sampling seed, category definitions. Regenerated dataset invalidates component-level data — check manifests.
   - **No action**: logging, comments, UI, infra

2. **Dataset review** — check coverage. If new categories needed, follow Phase 3 (preview gate included) for affected scope only.

3. **Update eval code** as needed.

4. **Run** — execute `run.<ext>`. Paired t-test vs previous. Update `doc.md` sections 6a and 6b. Output before/after comparison (overall ± CI, per category × difficulty, p-value, effect size). Flag regressions only if statistically significant at p < 0.05.

---

## Key Principles

- **Run first, ask second** — show outputs before asking questions; users react better than they generate from scratch
- **Skip everything** — every gate is optional; never hold the user at a step
- **Assertions over reference answers** — check criteria, not exact output; many valid responses can satisfy the same assertions
- **Grade outcomes, not paths** — don't check tool call sequence; agents find valid approaches designers didn't anticipate
- **Concise code** — readable, minimal, no unnecessary abstractions
- **Multiple trials for E2E** — run each case multiple times, report mean; component evals single-run
- **Always report uncertainty** — every score includes 95% CI
- **Reproducible generation** — every LLM call records model, template, seed
- **Full citation** — every result in `doc.md` references its run file, dataset, git hash
- **Single entry point** — `run.<ext>` runs everything
- **Judge model must be stronger** — at least one capability tier above production; set as `JUDGE_MODEL` constant; update when production model upgrades
- **Checkpoint and resume** — skip completed cases on re-run; timestamped results never overwrite each other
- **Study before asking** — read the codebase thoroughly; ask only about genuine gaps the code cannot answer
- **Plain-language translation** — after every set of scores, one sentence on what the number means in practice

---

## Reference Files

- `references/doc_template.md` — canonical structure for `doc.md`
- `references/evalcase_schema.md` — EvalCase schema, difficulty axes, assertions
- `references/augmentation_techniques.md` — augmentation technique catalog (load at Phase 3 step 8)
- `references/graders.md` — grader types, assertions vs rubrics, calibration, anti-patterns (load at Phase 2)
- `references/transcript_tool.md` — transcript viewer, annotation, test case seeding (load at Phase 5)
