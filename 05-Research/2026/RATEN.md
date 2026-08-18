## RATEN: an efficient robustness analysis and test enhancement framework for state machines

### Abstract
Modern software systems must operate flawlessly even when faced with unexpected runtime conditions like malformed messages, missing inputs, or protocol violations. Traditional verification methods fall short in evaluating how systems handle these deviations, leaving critical gaps in robustness analysis. RATEN, a novel framework for state machine-based systems, bridges this gap by introducing a quantitative approach to assess resilience. The article explores how RATEN computes two key metrics: offtrack cost, which measures the immediate impact of unexpected failures, and backtrack cost, which evaluates the effort required to recover to normal operation. By integrating property models as ground truth, RATEN automates robustness analysis and reduces dependency on manual expert input, addressing long-standing challenges in testing efficiency and scalability. The framework targets three common failure types—wrong messages, wrong payloads, and missing messages—and demonstrates its effectiveness through comprehensive evaluations on systems ranging from simple state machines to complex distributed architectures with over 2,300 states. Readers will discover how RATEN’s cost-based metrics enable fine-grained prioritization of testing efforts, its seamless integration with existing model-based testing tools, and its potential to enhance system reliability in dynamic environments. The article also highlights practical applications in industries like telecommunications and automotive, where robustness is non-negotiable, and provides insights into the framework’s performance, limitations, and future research directions.



**Full title:** RATEN: an efficient robustness analysis and test enhancement framework for state machines **Authors:** Majid Babaei (UFV), Yann-Gaël Guéhéneuc (Concordia) **Venue:** Software and Systems Modeling (Springer), accepted Feb 2026 **DOI:** 10.1007/s10270-026-01371-z **Repo:** https://github.com/ai4se4ai-lab/RATEN-ra4xstate

---

## TL;DR

RATEN is a **runtime robustness analysis framework** for finite-state-machine (FSM) systems. Instead of just checking "does the implementation match the spec?", it asks **"can the system stay acceptable when the environment misbehaves?"** It does this by comparing execution traces of a real implementation (Behavioral Model) against a hand-written requirements model (Property Model) that marks states as Good/Bad, and computes a **quantitative cost** for every deviation and every recovery. It also plugs into regression testing (MRegTest) to shrink test suites by ~47% on average without losing fault-detection power.

This extends the authors' earlier tool **ra4xstate** (2023) with quantitative cost, property-model querying for test filtering, a bigger evaluation, and open-source tooling on top of **XState**.

---

## Problem It Solves

Traditional approaches to robustness testing have 3 gaps:

1. **Manual effort** — someone has to hand-specify every possible environmental deviation and expected response.
2. **Test suite bloat** — hard to maintain, especially for distributed reactive systems.
3. **No quantitative signal** — existing tools give binary robust/non-robust verdicts, so you can't prioritize fixes or compare designs.

## Core Idea

- **Behavioral Model (BSM):** the actual implementation as an FSM.
- **Property Model (PSM):** a second FSM, written by domain experts, that classifies every reachable state as **Good** or **Bad** (ground truth for "acceptable behavior").
- Replay real execution traces against both models. Whenever the BSM takes an unexpected step (message received that the PSM didn't expect), classify the transition and assign it a **cost**:
    - **OTcost (Off-the-Track cost):** cost of the deviation itself (Good→Bad transition, or continued Bad→Bad).
    - **BTcost (Back-to-Track cost):** cost of the minimum-cost path to recover from Bad back to Good (computed via bounded BFS over the property model).
- If total cost > a user-defined threshold → system is labeled **"NotRobust"** for that trace.

### Three failure types studied (CRFs = Common Robustness Failures)

|Type|Description|Example|
|---|---|---|
|**WM – Wrong Message**|Unexpected message type at a state|Getting `cancel` when `confirm` is expected|
|**WP – Wrong Payload**|Right message, malformed/out-of-range data|Temperature reading of −500°C|
|**MM – Missing Message**|Expected message never arrives in time|Timeout / dropped packet|

### Transition classification (property model preprocessing)

- **L1:** Good → Good (fine, cost 0)
- **L2:** Good → Bad (violation — compute both OTcost & BTcost)
- **L3:** Bad → Good (recovery — BTcost only)
- **L4:** Bad → Bad (continued degradation — accumulate OTcost, recompute BTcost)

### Formal backbone

- FSM = ⟨S, s0, M, V, T, A⟩ — states, initial state, messages, variables, transitions, actions.
- Execution proceeds via **run-to-completion (rc) steps**, atomic and non-interruptible (standard reactive-systems assumption, aligned with SCXML/UML-RT).
- 3 well-formedness conditions required: deterministic transitions, transition completeness, and BSM/PSM alphabet consistency.

## Architecture (the "SRA" pipeline)

`ENV → Behavioral Model → traces → MReplayer → OTcost/BTcost calculators → ComputeCost → Robust / NotRobust verdict`

Key components:

- **MReplayer** — deterministic trace replay engine reusing prior work ([18],[19]) — regenerates variable values/states from minimal traces (rather than storing full state at every step).
- **OT/BT Cost Calculators** — implement Algorithm 2 (main cost loop) and Algorithm 3 (BFS-based recovery-cost search, capped by `depthMAX`).
- **Property Model Querying (Algorithm 4)** — feeds into **test suite reduction**: instead of relying on a domain expert to define "critical variables" for regression testing, RATEN auto-detects which test cases can reach a Bad state and filters test suites down to those.

## Integration: MRegTest (model-based regression testing)

RATEN extends MRegTest's `TestcaseReduction` module — mapping "critical variable" bad-values to Property Model Bad states, so test-case filtering becomes automatic and stays in sync as requirements evolve (no more manual re-specification).

## Experimental Setup

- Implemented as an **XState** extension (JS/React 17).
- 8 case studies (4 simple, 4 "instrumented"/debug-enhanced versions), from 7 states / 12 transitions (Content Management) up to **2,364 states / 2,396 transitions** (Refined FailOver system).
- 100,000 traces per case study (main experiments), 500,000 traces for overhead comparison, 20 repetitions each.
- Testing strategies: **Basic** (1 CRF injected) vs **Compound** (multiple CRFs).
- Execution modes: **Single**, **Sequential** (recover between failures), **Nested** (new failure before recovery completes — worst case).

## Key Results

|Metric|Result|
|---|---|
|**Precision**|0.77 – 1.00 across all scenarios|
|**Recall**|0.78 – 1.00 across all scenarios|
|**Analysis time (largest model, worst mode)**|≤ 38.23 sec even for the 2,364-state model|
|**Runtime overhead vs. trace-annotation baseline**|1.02× – 1.28× (acceptable)|
|**Test suite size reduction (avg.)**|~47% across CRF types/modes|
|— WM (wrong message)|17% / 62% / 59% (Single/Seq/Nested)|
|— WP (wrong payload)|19% / 37% / 78%|
|— MM (missing message)|43% / 54% / 77% (biggest wins here)|
|**Execution time vs plain MRegTest**|Faster in Sequential/Nested (up to 54% faster); slightly slower in Single mode due to query overhead|

Notes on patterns:

- MM (missing message) scenarios are cheapest to analyze — timeout-based detection, less state exploration.
- Overhead grows with model complexity (biggest for FO/RFO) because more variables must be regenerated during replay.
- Nested-mode failures (cascading, no recovery time) are consistently the hardest to analyze and the most expensive.

## How It Compares to Related Work

- vs. **Zhang et al. [2] behavioral robustness** — theirs is more theoretical/hard to apply to real unmodeled environments; RATEN is concrete + replay-based.
- vs. **Henzinger et al. [8] / Tabuada et al. [32] (quantitative/cost-based verification)** — RATEN computes costs via **property-model-based trace replay** rather than abstract distance metrics or direct trace analysis.
- vs. **binary robustness/verification tools** — RATEN's key differentiator is the **quantitative cost model** (OTcost/BTcost) enabling prioritization, not just pass/fail.
- Builds directly on the authors' own **ra4xstate** (2023) and **MReplayer** (MODELS 2020) work.

## Assumptions / Scope (important caveats)

RATEN only applies to systems that satisfy:

1. **Message passing / strong encapsulation** (no shared state between components)
2. **Run-to-completion** execution (no interleaving mid-message)
3. **Deterministic** message handling (replayable)

This covers actor-based systems, SCXML, UML-RT, and many telecom/automotive/IoT reactive systems — but **excludes** systems with orthogonal/concurrent composite states, non-determinism, or shared-memory concurrency.

## Stated Limitations & Future Work

- Only 3 CRF types covered (WM/WP/MM) — ordering violations, duplicates, resource exhaustion not yet handled.
- State machine language restricted (no fork/join/history/final states, no orthogonal regions).
- Only integrated with XState + MRegTest so far — broader tool support would help adoption.
- Future directions: timing/resource-constraint failure models, ML-based automatic cost-threshold tuning, industrial case studies, formal theoretical foundations, CI/CD & cloud-native integration.

## Practical Takeaways (for reuse)

- **The OTcost/BTcost split is the reusable idea** — separating "how bad is this deviation" from "how hard is it to recover" is a clean general pattern for robustness/resilience metrics, portable beyond FSMs (e.g., could inspire similar cost-based thinking for API contract testing, chaos engineering scoring, or SRE error-budget style thinking).
- **Property-model-as-ground-truth for test filtering** is a nice trick to reduce reliance on tribal/expert knowledge in regression suites — worth considering for any project with a "critical variables" or "known bad state" concept in its test infra.
- Biggest ROI from this class of technique shows up in **Nested/Sequential failure modes** — i.e., systems experiencing repeated or overlapping faults — much more than isolated single-failure testing. Relevant if you're prioritizing resilience testing investment.
- Overhead (1.02–1.28×) is presented as an argument that trace-replay-based robustness analysis is "cheap enough" to bolt onto existing CI without major infra cost — useful data point if pitching similar tooling internally.

## Citation

Babaei, M., Guéhéneuc, Y.-G. (2026). RATEN: an efficient robustness analysis and test enhancement framework for state machines. _Software and Systems Modeling_. https://doi.org/10.1007/s10270-026-01371-z