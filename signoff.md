# Day 2 Signoff (Asker)

## Did the explainer close my gap?
Yes. The explainer closed my gap at mechanism level.

## My original gap
I could not defend how the judge score is coupled to generator behavior at inference time, and I could not clearly distinguish rejection sampling, best-of-N, and reranker in my own pipeline.

## What became clear
- A scalar judge score is not a strategy by itself; selection policy is the strategy.
- Rejection sampling, best-of-N, and reranker differ in stopping rule, cost profile, and latency behavior.
- I can now explain which metadata must be logged to make Delta A reproducible across collaborators.

## Evidence that the gap is closed
- I can now state the decision mechanism in plain language.
- I can describe cost formulas for each strategy.
- I can point to concrete artifact edits needed for reproducibility.

## Remaining confusion
No blocker remains for Day 2. Future depth area: strategy behavior under stricter latency SLOs.

## Asker confirmation
- Name: Bethel
- Date: 2026-05-06
- Status: Explainer accepted

s