# Explainer for Bethel — Week 12 Day 1
## Agent/Judge Coupling at Inference Time: Rejection Sampling vs Best-of-N vs Reranker

## 1) The gap this explainer closes

Your judge returns a scalar score, but the score alone does not define behavior.  
The missing mechanism is the **selection policy** that converts judge scores into the next action.

This is the exact undefended step in your current pipeline:

- `scoring_evaluator.py` scores candidate outputs
- the agent chooses one output (or keeps generating)
- but the strategy is implicit, not documented

Because strategy is implicit, your Delta A claim in `ablation_results.json` is not fully reproducible by another FDE.

---

## 2) The three strategies at code level

### A) Rejection Sampling

**Mechanism:** Generate one candidate, score it, accept only if score >= threshold; otherwise generate again.

```python
def select_rejection_sampling(prompt, generator, judge, threshold=0.8, max_tries=8):
    for t in range(max_tries):
        y = generator(prompt)
        s = judge(prompt, y)
        if s >= threshold:
            return y, {"strategy": "rejection_sampling", "tries": t + 1, "score": s}
    return y, {"strategy": "rejection_sampling", "tries": max_tries, "score": s, "fallback": True}
```

**What changes behavior:** threshold and max_tries.

**Failure mode:** silent cost blow-up if threshold is too strict (low acceptance rate).

---

### B) Best-of-N

**Mechanism:** Generate exactly N candidates, score all N, pick the top score.

```python
def select_best_of_n(prompt, generator, judge, n=3):
    cands = [generator(prompt) for _ in range(n)]
    scored = [(c, judge(prompt, c)) for c in cands]
    best, best_score = max(scored, key=lambda x: x[1])
    return best, {
        "strategy": "best_of_n",
        "n_candidates": n,
        "scores": [s for _, s in scored],
        "best_score": best_score,
    }
```

**What changes behavior:** N and candidate diversity.

**Failure mode:** fixed cost floor of N generator calls even when first draft is already strong.

---

### C) Reranker (Two-stage)

**Mechanism:** Build candidate set in stage 1 (possibly cheaper model/search), then judge scores candidates in stage 2 and selects top-1.

```python
def select_reranker(prompt, candidate_fn, judge, n=6):
    cands = candidate_fn(prompt, n=n)   # candidate_fn can be cheaper or structured retrieval
    scored = [(c, judge(prompt, c)) for c in cands]
    best, best_score = max(scored, key=lambda x: x[1])
    return best, {
        "strategy": "reranker",
        "n_candidates": len(cands),
        "scores": [s for _, s in scored],
        "best_score": best_score,
    }
```

**What changes behavior:** candidate quality/diversity in stage 1 and judge strictness in stage 2.

**Failure mode:** reranker cannot recover if candidate set is weak.

---

## 3) Cost formulas (the load-bearing tradeoff)

Let:
- `C_gen` = average cost of one generator call
- `C_j` = average cost of one judge call
- `N` = candidate count
- `p_accept` = probability score >= threshold in rejection sampling
- `E[K]` = expected tries before acceptance

### Rejection sampling

- `E[K] ≈ 1 / p_accept` (bounded by `max_tries`)
- `Expected cost ≈ E[K] * (C_gen + C_j)`
- **Unbounded tendency** if threshold too high and no strict max_tries

### Best-of-N

- `Cost = N*C_gen + N*C_j`
- Predictable, linear, easy to budget

### Reranker

- `Cost = C_candidate_stage + N*C_j`
- Good if candidate stage is cheap and diverse

---

## 4) How to identify what your current Tenacious agent is actually doing

Use this one-function audit pattern:

```python
def audit_strategy(meta):
    # meta = per-request decision metadata you log
    # Expected keys differ by strategy
    if "tries" in meta and meta.get("strategy") == "rejection_sampling":
        return "rejection_sampling"
    if meta.get("strategy") == "best_of_n" and "n_candidates" in meta:
        return "best_of_n"
    if meta.get("strategy") == "reranker" and "n_candidates" in meta:
        return "reranker"
    return "unknown_or_implicit"
```

If your logs do not contain:
- `decoding_strategy`
- `n_candidates` or `tries`
- threshold/max_tries (if applicable)

then your strategy is effectively implicit and not reproducible.

---

## 5) Applying this to your Tenacious artifacts

### Artifact 1: `scoring_evaluator.py`

Add an explicit strategy wrapper:

```python
def select_output(prompt, generator, judge, strategy="best_of_n", n_candidates=3, threshold=0.8, max_tries=8):
    if strategy == "best_of_n":
        return select_best_of_n(prompt, generator, judge, n=n_candidates)
    if strategy == "rejection_sampling":
        return select_rejection_sampling(prompt, generator, judge, threshold=threshold, max_tries=max_tries)
    raise ValueError(f"Unsupported strategy: {strategy}")
```

This makes strategy explicit, testable, and swappable.

### Artifact 2: `ablation_results.json`

Add these fields to each ablation row:
- `decoding_strategy`
- `n_candidates` (for best-of-N/reranker)
- `threshold`, `max_tries` (for rejection sampling)

Example:

```json
{
  "delta_a": {
    "score_baseline": 0.4848,
    "score_trained": 0.8633,
    "delta": 0.3785,
    "95_ci": [0.3487, 0.4084],
    "p_value": 0.0,
    "n_tasks": 67,
    "decoding_strategy": "best_of_n",
    "n_candidates": 3
  }
}
```

### Artifact 3: `model_card.md` (Intended Use + Evaluation Results)

State exactly:
- strategy used during held-out eval
- strategy hyperparameters (`n_candidates` or threshold/max_tries)
- warning that changing strategy changes cost/latency/quality

---

## 6) Cost-quality recommendation for Tenacious ($10 envelope, ~200-task held-out)

Given your Week 11 constraints, the safest default is:

**Best-of-N with small N (2 or 3)** for primary eval runs.

Why:
1. Bounded and predictable cost.
2. Easy reproducibility across collaborators.
3. Often strong quality lift vs single candidate.
4. Lower operational risk than rejection sampling threshold misconfiguration.

Use rejection sampling only when:
- threshold is calibrated on dev,
- max_tries is hard-capped,
- acceptance-rate monitoring is live.

---

## 7) Hands-on evaluation protocol (small but decisive)

Run 3 conditions on the same held-out slice and same seed:

1. `strategy=best_of_n, n=1` (single-pass baseline)
2. `strategy=best_of_n, n=3`
3. `strategy=rejection_sampling, threshold=t, max_tries=5`

Log:
- score mean
- cost/task
- p50 latency
- acceptance rate (for rejection sampling)

This gives the exact curve needed for a defendable deploy decision.

---

## 8) Scope discipline: what is intentionally out-of-scope

Not needed to close this question:
- speculative decoding internals
- beam-search theory
- token-level entropy diagnostics

Those can improve future optimization, but do not block your immediate reproducibility and reporting gap.

---

## 9) Bottom line

A judge score is not the strategy.  
Your **selection policy** is the strategy.

If you do not explicitly log and publish that policy, your Delta A claim is not reproducible.  
For Tenacious right now: make strategy explicit in code and artifacts, start with best-of-3, and report cost-quality with strategy metadata attached.

