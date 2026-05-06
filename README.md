# Knowledge Gap Formulation Pair - Day Two

This repository contains Day Two writing artifacts for a Bethel knowledge-gap
exercise focused on **agent/judge coupling at inference time**.

The core problem documented here is:
- A judge model returns a scalar score for candidate generations
- The exact selection mechanism between that score and the next agent action is
  often left implicit
- That hidden mechanism affects quality, latency, and reproducibility

The materials compare and explain three inference-time strategies:
- Rejection sampling
- Best-of-N
- Reranking

## What this repo is

A documentation-first workspace that captures:
- The concrete technical question being asked
- Grounded references to artifacts and reproducibility concerns
- Explanations, summaries, and communication-ready drafts

## What it does

It helps turn an ambiguous implementation detail into a defendable, reproducible
technical claim by:
- Defining the mechanism-level gap
- Mapping the gap to specific artifacts that should be updated
- Explaining cost-quality tradeoffs of competing decoding strategies
- Producing briefings for team communication (morning/evening summaries,
  signoff-ready text)

## Main files

- `question.md`: Primary diagnostic question and why it matters
- `explainer.md`: Detailed technical explanation of the three strategies
- `source.md`: Supporting context and source grounding material
- `grounding_commit.md`: Grounding notes tied to artifact-level changes
- `morning_call_summary.md`: Brief for morning sync
- `evening_call_summary.md`: End-of-day summary
- `signoff.md`: Signoff-oriented summary
- `theard.md`, `x.md`: Additional working drafts/notes

## Intended use

Use this repo as a reference packet for:
- Internal technical discussions
- Reproducibility documentation updates
- Model card/evaluation writeups where inference-time strategy must be explicit
