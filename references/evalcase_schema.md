# EvalCase Schema

Every entry in `data/cases.jsonl` must conform to this schema. Every `model` field must reference a real model.

```
{
  id:               string,                     # stable unique ID (e.g. uuid or hash)
  input:            string | object,            # string for text, JSON-encoded object for structured inputs
  is_real:          bool,                       # true = production, false = synthetic — see integrity note below
  categories:       List[string],              # multi-label, based on input characteristics
  assertions:       List[string],              # per-case criteria this response must meet
  difficulty: {
    rationale:          string,
    model:              string,
    prompt_template:    string,
    label:          "easy" | "medium" | "hard",
    axes: {
      clarity:      1-3,                        # 1=unambiguous signals, 3=ambiguous or conflicting signals
      reasoning:    1-3,                        # 1=single-step, 3=multi-step or cross-source
      domain:       1-3                         # 1=common knowledge, 3=expert or domain-specific
    },
  },
  variants: [{                                  # augmented versions of this case
    input:                         string | object,
    technique:                     string,
    technique_selection_rationale: string,
    model:                         string,
    prompt_template:               string,
    generation_seed:               int
  }],
  metadata: Any
}
```

## is_real Integrity

`is_real: true` means the input was genuinely sourced from production — not hand-authored, not paraphrased, not retroactively attributed. If production data closely matches hand-authored cases (same phrasing, same ids, same scenarios), it is not real production data — keep `is_real: false`. Mislabelling inflates the apparent production coverage and corrupts lineage tracking.

## Assertions

`assertions` are specific, per-case criteria the agent's response must meet for this task. The skill drafts these from understanding the task during Phase 3; refine with the user before running.

Examples (drawn from different domains to illustrate the pattern):
- "The agent ran the failing tests before submitting the patch" (coding agent)
- "The answer cites at least one source returned by the retrieval tool" (RAG agent)
- "The agent did not take an irreversible action before confirming with the user" (any high-stakes agent)
- "The reply addresses all sub-questions in the user's message" (general assistant)

Assertions check criteria, not exact output. Many valid responses can satisfy the same assertions. Do not write expected outputs — there are always multiple correct ways to respond.

## Suite-Level Rubrics

Generic quality dimensions (empathy, clarity, groundedness) are **not** stored in EvalCase. They live in the suite config and are applied to every case. See `references/graders.md`.

## Difficulty Axes

| Axis | 1 | 2 | 3 |
|------|---|---|---|
| **clarity** | unambiguous signals | some ambiguity | ambiguous or conflicting signals |
| **reasoning** | single-step | moderate | multi-step or cross-source |
| **domain** | common knowledge | some specialisation | expert or domain-specific |

`label` is assigned holistically by the LLM using the axes as guidance, not mechanically derived.
