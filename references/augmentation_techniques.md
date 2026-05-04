# Augmentation Techniques

Use these techniques when generating augmented variants from seed cases (Phase 3, step 6). Select 3–4 techniques per seed. Not all techniques apply to all input types — choose based on the system's domain and what failure modes the eval is designed to surface.

| Technique | What it does | What it tests |
|-----------|--------------|---------------|
| **Surface variation** | Vary phrasing, format, order, or style while keeping content the same | Robustness to presentation differences |
| **Entity / content swapping** | Change names, topics, entities, or data values while preserving structural characteristics | Generalisation beyond seed content |
| **Signal dilution** | Reduce the clarity of the relevant signal (bury key content, add noise, make cues implicit) | Handling of weak or obscured signals; increases difficulty |
| **Polarity / edge cases** | Introduce inputs at decision boundaries (borderline keep/discard, barely relevant, mixed signals) | Calibration at the margin |
| **Composition** | Combine characteristics from two categories in a single input | Handling of mixed signals or multi-step reasoning |
| **Volume / scope variation** | Vary input size or breadth (larger document sets, more sources, more items) | Scaling behaviour |
| **Noise injection** | Introduce realistic noise: formatting errors, missing fields, malformed data | Robustness to real-world imperfection |

## Guidance

- **Maintain balanced difficulty across augmented variants.** Re-score difficulty for each generated variant using the axes in `evalcase_schema.md`.
- **Technique selection is LLM-driven.** The generation script selects techniques per seed via LLM call; log the selection rationale and prompt template in the `variants` array.
- **Target ~50 cases per category** across seeds + augmentations.
- **Aim for ~3–4 techniques per seed.** Fewer risks underrepresenting failure modes; more can produce redundant cases.
