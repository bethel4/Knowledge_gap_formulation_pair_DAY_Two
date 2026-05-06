# Week 12 — Day 2 Question

## The Question

My Tenacious agent uses a trained judge that returns a scalar score after each
generation step. But I cannot defend what happens between that score and the
agent's next action at the mechanism level.

**Specifically: what is the difference between rejection sampling, best-of-N,
and a reranker as inference-time decoding strategies for coupling a judge score
to a generator — which one does my agent actually use, and what does the
cost-quality tradeoff of each look like measured against my Tenacious-Bench
held-out eval?**

---

## Why This Question Is Diagnostic

This is not a question about decoding strategies in the abstract. It names one
specific undefended step in my production pipeline:

- My `scoring_evaluator.py` returns a scalar score per candidate output
- My agent selects among candidates based on that score
- I cannot name the selection mechanism — whether it is rejection sampling
  (generate until score exceeds threshold), best-of-N (generate N in parallel,
  pick highest), or a reranker (score all candidates, return top-1)
- Each of these has a different cost profile, latency profile, and failure mode
- My `ablation_results.json` reports a Delta A lift number but does not
  document which selection mechanism produced it — meaning the number is
  not reproducible by a colleague who wires the judge differently

The mechanism I cannot currently defend: **how a scalar reward signal from a
trained judge couples back to token generation at inference time, and what the
three main coupling strategies actually do differently at the code level.**

---

## Grounding in My Artifacts

This question requires concrete edits to three specific artifacts:

| Artifact | What changes if the gap closes |
|---|---|
| `scoring_evaluator.py` | Add an explicit `decoding_strategy` parameter with documented default, replacing the implicit library default currently in use |
| `ablation_results.json` | Add a `decoding_strategy` field to every ablation row so the Delta A number is reproducible |
| `model_card.md` — "Intended Use" section | Name the decoding strategy the judge was evaluated with, so a downstream user does not rewire it and get different numbers |

Specific claim that needs defending, from my current `ablation_results.json`:

> Delta A: +X.X pp on Tenacious-Bench held-out (95% CI: [...], p < 0.05)

This number was produced by some coupling mechanism between the judge and the
generator. I do not have that mechanism documented. A colleague cannot reproduce
this number without knowing which decoding strategy was used — and neither can I
with full confidence six months from now.

---

## Why This Is Generalizable

Every FDE who ships a trained judge faces this exact decision point. It applies to:

- Any Path B deployment where a preference-trained critic is used as a
  rejection-sampling or rollback layer in front of a generator
- Any Path A deployment where best-of-N is used at inference to lift
  generation quality without retraining
- Any production agent where cost and latency constraints force a choice
  between generating one output and scoring it vs. generating N and picking
- The tradeoff is not theoretical — rejection sampling can 3x your inference
  cost silently if the threshold is set wrong, and best-of-N has a hard
  cost floor of N × single-inference-cost regardless of score

This is not a Tenacious-specific question. It is the decision every FDE makes
when they wire a judge to an agent, and most make it by accepting the library
default without knowing what it is.

---

## Why This Is Resolvable in One Explainer

A thoughtful colleague can close this gap with a 600–1,000 word blog post that:

1. Names the three strategies (rejection sampling, best-of-N, reranker) and
   explains what each does at the code level with a minimal working example
2. Shows the cost formula for each: rejection sampling is unbounded,
   best-of-N is N × cost, reranker is N × cost + scoring overhead
3. Shows the quality curve for each on a small benchmark — where they
   diverge and where they converge
4. Gives a one-function audit that lets an FDE check which strategy their
   agent is currently using
5. Makes a concrete recommendation for the Tenacious use case given the
   $10 compute envelope and the 200-task held-out eval

This does not require a textbook chapter. It requires someone who has read
the best-of-N or speculative decoding literature, run a real inference
experiment, and looked at the cost log on both sides of the tradeoff.

---

## Artifact Pointer

Primary artifact: `ablation_results.json` — Delta A row
Secondary artifacts: `scoring_evaluator.py`, `model_card.md`
Trace source: `held_out_traces.jsonl` from Act IV

Knowing the answer lets me commit two concrete edits:

1. Add `decoding_strategy: "best_of_n"` (or whichever is correct) with
   `n_candidates: X` to every row in `ablation_results.json`

2. Add a `select_output(candidates, judge, strategy="best_of_n")` wrapper
   function to `scoring_evaluator.py` that makes the strategy explicit,
   testable, and swappable
