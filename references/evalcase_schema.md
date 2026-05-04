# EvalCase Schema

Every entry in `data/cases.jsonl` must conform to this schema. Every `model` field must reference a real model — static hard-coded cases with no LLM-scored difficulty or augmentation are not valid.

```
{
  id:               string,                     # stable unique ID (e.g. uuid or hash)
  input:            string | object,            # serialised input — string for text, JSON-encoded object for structured inputs
  is_real:          bool,                       # true = production, false = synthetic
  categories:       List[string],              # multi-label, based on input characteristics
  difficulty: {
    label:          "easy" | "medium" | "hard",
    axes: {
      clarity:      1-3,                        # 1=unambiguous signals, 3=ambiguous or conflicting signals
      reasoning:    1-3,                        # 1=single-step, 3=multi-step or cross-source
      domain:       1-3                         # 1=common knowledge, 3=expert or domain-specific
    },
    rationale:          string,
    model:              string,
    prompt_template:    string
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

## Difficulty Axes

| Axis | 1 | 2 | 3 |
|------|---|---|---|
| **clarity** | unambiguous signals | some ambiguity | ambiguous or conflicting signals |
| **reasoning** | single-step | moderate | multi-step or cross-source |
| **domain** | common knowledge | some specialisation | expert or domain-specific |

`label` should be assigned holistically by the LLM using the axes as guidance, not mechanically derived.
