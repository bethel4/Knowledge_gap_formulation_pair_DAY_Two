# Sources — Failure Attribution in LLM Agent Systems

## Core Paper (Primary Reference)

* Zhang et al., **“Which Agent Causes Task Failures and When? On Automated Failure Attribution of LLM Multi-Agent Systems”** (ICML 2025)

  * Introduces the *Who&When* dataset
  * Defines **decisive error (agent, step)**
  * Evaluates attribution methods (all-at-once, step-by-step, binary search)
  * Key insight: attribution is hard, especially step-level (≈14% accuracy)

---

## Agent & Tooling Foundations

* Hong et al., 2023 — Multi-agent LLM collaboration frameworks
* Wu et al., 2023 — Agent interaction protocols
* Li et al., 2023 — Tool-augmented LLM systems

**Why relevant:**
Defines how agents operate (turn-based, tool-using), which is the structure you’re debugging.

---

## LLM-as-a-Judge (Critical for Logging + Attribution)

* Gu et al., 2024–2025 — *Survey on LLM-as-a-Judge*
* Zheng et al., 2023 — Using LLMs for evaluation

**Key idea:**
Use LLMs not just to act, but to **evaluate and attribute failure** — exactly what your logging layer needs.

---

## Process-Level Supervision (Step Attribution)

* Lightman et al., 2023 — *Let’s Verify Step by Step (PRM)*
* Shao et al., 2024 — *DeepSeek-Math (process reward models)*

**Why relevant:**
Your problem = **step-level failure detection**
These works show how to:

* score intermediate steps
* detect where reasoning breaks

---

## Synthetic Data & Benchmark Construction

* Liu et al., 2024 — *Best Practices on Synthetic Data*
* Xu et al., 2024 — *Magpie (self-generated instruction data)*

**Why relevant:**
You often don’t have labeled failures → must **generate + filter failure cases**.

---

## Dataset Design & Evaluation Standards

* Gebru et al., 2021 — *Datasheets for Datasets*
* Pushkarna et al., 2022 — *Data Cards*

**Why relevant:**
Your failure logs = a dataset
These define:

* how to structure it
* how to document failure taxonomy

---

## Benchmark Reliability & Contamination

* Chen et al., 2025 — *Dynamic Evaluation & Contamination*

**Why relevant:**
Failure attribution must be:

* unbiased
* not overfit to training traces

---

## Preference Learning (Failure Judging Models)

* Rafailov et al., 2023 — *DPO*
* Meng et al., 2024 — *SimPO*
* Hong et al., 2024 — *ORPO*
* Kim et al., 2024 — *Prometheus 2 (judge models)*

**Why relevant:**
Train a model to:

* distinguish good vs bad outputs
* detect failure types automatically

---

## Failure Modes in Agent Systems (Conceptual)

* Shinn et al., 2024 — Self-reflection in agents
* Huang et al., 2023 — LLM reasoning failures

**Key categories (maps to your pipeline):**

1. **Reasoning failure** → planner mistake
2. **Execution failure** → wrong tool args
3. **Environment failure** → API/runtime

---

## What You Should Take From These Sources

From all of the above, the *practical system design* becomes:

### 1. Treat failures as (agent, step)

From Zhang et al.
→ Always log **who acted + when**

### 2. Separate failure layers

From agent/tool papers
→ Planner vs Executor vs External

### 3. Add step-level evaluators

From PRM / LLM-as-judge
→ Detect failure *during execution*, not just at the end

### 4. Build a failure dataset

From synthetic data + datasheets
→ Your logs become training + evaluation data

### 5. Train a failure classifier/judge (optional but powerful)

From DPO / Prometheus
→ Automate attribution instead of manual debugging

---

## If You Only Remember 3 Things

1. **Log structure matters more than model choice**
2. **Step-level attribution is the real bottleneck**
3. **Without separating reasoning vs tool vs runtime, your metrics are misleading**

---

## Suggested Next Step (Practical)

Implement this minimal logging schema:

```json
{
  "trace_id": "...",
  "step": 12,
  "agent": "planner",
  "action": "call_api",
  "tool_args": {...},
  "result": {...},
  "error_type": "reasoning | tool | runtime | none",
  "confidence": 0.82
}
```

Then:

* run ablations per error_type
* measure cost per failure type
* prioritize fixes based on frequency × cost

---

This is exactly how you move from:
**“the system failed” → “this specific step, by this agent, for this reason caused it.”**
