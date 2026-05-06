# Day 2 Grounding Commit

## Purpose
This file records what I (the asker) changed in real project artifacts after the explainer closed my gap.

## Gap addressed
Inference-time coupling between judge score and generator action was implicit and not reproducible.

## Artifacts updated (or queued for immediate update)

1. `scoring_evaluator.py`
- Added/plan to add explicit selection wrapper:
  - `select_output(..., strategy=..., n_candidates=..., threshold=..., max_tries=...)`
- Removed implicit strategy behavior from hidden defaults.

2. `ablation_results.json`
- Added/plan to add strategy metadata per ablation row:
  - `decoding_strategy`
  - `n_candidates` (when applicable)
  - `threshold` and `max_tries` (for rejection sampling)

3. `model_card.md` (Intended Use / Evaluation Results)
- Added/plan to add explicit note on strategy used during held-out evaluation.
- Added caution that changing strategy changes cost/latency/quality and can invalidate direct comparison.

## Why this improves artifact quality
- Makes Delta A claim reproducible.
- Lets another FDE replicate the same inference policy.
- Prevents silent drift caused by library defaults.

## Commit note
These changes are grounded directly in the Day 2 explainer and evening-call agreement.

## Asker
- Name: Bethel
- Date: 2026-05-06

