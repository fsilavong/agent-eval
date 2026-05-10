# Graders

## Metrics vs Graders

These are distinct concepts:

- **Metrics** are what you care about — the dimensions of quality you track across runs (e.g. task completion rate, tone quality). Each component has one **primary metric** used for regression decisions, and optionally a small number of **diagnostic metrics** that explain why the primary moved.
- **Graders** are how you measure them — the implementation that produces a score (assertions, rubric calls, code checks). Multiple graders can feed into a single metric.

Metric selection happens in Phase 2. Grader implementation happens in Phase 4.

---

Three types of graders, used in combination. For each task, scoring can be weighted (combined scores hit a threshold), binary (all must pass), or hybrid.

## Code-Based

Fast, cheap, objective, reproducible. Use as the first line where possible.

**Before implementing any code check, verify it passes on at least one known-good example from the codebase.** If a check conflicts with the agent's actual design (e.g. the agent always calls tool A before tool B, but the check requires B before A), surface the contradiction to the user before proceeding — do not silently penalise behaviour that may be correct.

- **Tool call verification** — check which tools were called, with what params, and what the response was. Tool responses in the transcript cover most state verification: if a call succeeded or failed, it's visible there.
- **String match** — exact, regex, or fuzzy match on output text
- **Static analysis** — lint, type checks, security scanners (for coding agents)
- **Transcript analysis** — n_turns, n_tool_calls, n_total_tokens, max_turns constraint

## Model-Based

### Per-Case Assertions

Each EvalCase has an `assertions` list — specific criteria that this case's response must meet. Checked per case with a targeted yes/no LLM call per assertion (one call per assertion, not bundled).

Examples (drawn from different domains to illustrate the pattern):
- "The agent ran the failing tests before submitting the fix" (coding agent)
- "The summary includes all three key findings from the retrieved documents" (research agent)
- "The booking was made within the user's stated availability window" (scheduling agent)
- "The reply does not recommend an action that the retrieved policy prohibits" (any policy-grounded agent)

**The skill drafts assertions from the codebase** during Phase 3. Show to the user for refinement — don't ask them to write from scratch. Assertions check criteria, not exact output. Many valid responses can satisfy the same assertions.

### Suite-Level Rubrics

Generic quality dimensions applied to every case in the suite. Defined once in the suite config, not per-case. Grade each dimension with a separate LLM call.

Examples (adapt dimensions to the system under test):
- Groundedness: are claims in the reply supported by retrieved or tool-returned data?
- Clarity: is the outcome or next step stated plainly without ambiguity?
- Appropriateness: does the tone and scope match the task context (professional support, technical peer, concise assistant, etc.)?

Draft rubrics during Phase 2 and present to the user for sign-off.

**Calibration**: before running at scale, test each rubric on three cases — a clearly good output, a clearly bad one, and a borderline one. Show results to the user. Adjust until they agree. Re-calibrate periodically using transcript tool annotations as accumulated ground truth.

## Human

Via the transcript tool (`references/transcript_tool.md`). Human annotations are the reference standard — use them to validate and calibrate model-based graders over time, not as the primary grading method.

## Anti-Patterns

- **Don't check tool call sequence** — agents find valid approaches eval designers didn't anticipate. Grade outcomes and assertions, not specific paths.
- **Don't bundle grading into one LLM call** — grade each dimension in isolation.
- **Don't reference-answer match** — assertions check criteria, not exact output.
- **0% pass rate = broken task or grader, not broken agent** — flag loudly and redirect to transcript tool before drawing conclusions.
