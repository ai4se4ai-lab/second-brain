
**Type:** Full paper (8 pages) | **Venue:** SERP4IoT '24 — 2024 ACM/IEEE 6th Int'l Workshop on Software Engineering Research & Practices for the Internet of Things (Lisbon, Apr 20, 2024) **Author:** Majid Babaei (McGill) — solo author **DOI:** 10.1145/3643794.3648273 **Code:** https://github.com/drmajidbabaei/fbpmn-iot-comm **Tags:** #AI4SE #formal-methods #BPMN #TLA+ #IoT #model-verification #SERP4IoT2024

---

## 1. One-line summary

Formalizes **13 IoT-specific communication models** (pull/push × Bag/Fifo/Causal/Synchronous exchange patterns) for BPMN 2.0, gives each a **TLA+ translation template**, and empirically verifies **soundness and safety** properties across 5 case studies of increasing complexity using the TLC model checker.

## 2. Problem it solves

- BPM systems are increasingly extended to model IoT-driven processes (sensors/actuators exchanging data), but existing BPMN-for-IoT extensions (uBPMN, BPMNE4WSN, BPMN4CPS, BPMNE4IoT, etc.) either:
    - don't formally define **communication semantics** at all, or
    - don't cover the **pull vs. push** message-exchange patterns that are fundamental to how IoT devices actually communicate (per the IoT Reference Model).
- Without formal semantics, IoT-BPMN models are ambiguous and hard to verify — you can't check properties like "does this process always terminate correctly" or "can a channel ever hold more than one undelivered token."
- Closest prior work (Houhou et al.'s time-sensitive BPMN collaboration semantics) formalizes communication models but **doesn't cover push/pull** — this paper's specific gap to fill.

## 3. Core contributions

1. **Formalization of a subset of BPMN 2.0 collaboration semantics** as a typed graph (nodes = tasks/gateways/events/processes, edges = sequence flows + message flows), with well-formedness conditions (C1–C11).
2. **13 communication models** for IoT-aware BPMN, combining pull/push exchange patterns with 4 ordering semantics (Bag, Fifo, Causal, Synchronous — Fifo further splits into pair/inbox/outbox/all).
3. **TLA+ translation templates** for every model, enabling automated formal verification (soundness + safety) via the TLC model checker.

## 4. Formal model — key building blocks

**BPMN Graph:** `G_b = ⟨N, E, M, R⟩` — nodes, edges, message types, containment relation (process ⊇ subprocess). Node types: `T_Nodes = A ∪ G ∪ E ∪ {P}` (Activities, Gateways, Events, Process). Edge types: `T_Edges = SF ∪ {MF}` (Sequence Flow ∪ Message Flow).

**Configuration** `γ = ⟨M_e, M_n, ξ⟩`:

- `M_e`, `M_n` = token markings on edges/nodes (natural numbers).
- `ξ = ⟨Γ, Θ⟩` = network state: `Γ` = the communication model in effect, `Θ` = pool of in-flight (sent-but-not-received) messages.
- Execution = a **trace** `T = {γ_0, γ_1, …}` of configurations, driven by start/complete predicates `sP`/`cP` per node and transmit/receive predicates `tP`/`rP` for messages.

**The 13 communication models** (4 sender/receiver processes p1,p2 → p3,p4 sending messages m1,m2 used as running example):

|Family|Pull variant(s)|Push variant(s)|Ordering guarantee|
|---|---|---|---|
|**Bag**|PLB|PSB|No ordering — arbitrary order, consumed in any order|
|**Fifo**|PLPF (pair), PLIF (inbox), PLOF (outbox), PLAF (all)|PSPF, PSIF, PSOF, PSAF|Increasingly strict: pair-wise → per-receiver → per-sender → **global** emission order|
|**Causal**|PLC|PSC|Causally-related messages (m1 ∝ m2) preserve causal order (vector-clock based)|
|**Synchronous**|SYNC|SYNC (identical for pull/push)|Transmit = receive, atomic — no buffering at all|

Pull = consumer fetches data when ready; Push = producer/subscriber-driven delivery; the paper notes Sync collapses the pull/push distinction since there's no buffering window.

**TLA+ encoding pattern:** each model gets a `net` variable (Bag / Sequence / function-of-sequences depending on ordering needs) plus `tP`/`rP` operators defining how sending/receiving mutate `net`. E.g. Bag models use TLA's `Bags` module with multiset union/difference; Fifo-all uses a plain sequence with `Append`/`Head`/`Tail`; Causal uses per-peer **vector clocks** to gate receive on no causally-earlier message being outstanding; Sync uses a singleton set that must be empty before a new send.

## 5. Verification setup

- **Properties checked:**
    - **Soundness** (option to complete, proper completion, no dead activities; collaboration-level: no undelivered messages).
    - **Safeness** (no sequence-flow edge ever holds >1 token, at process and collaboration level).
- **Tooling:** semantics + case studies encoded in TLA+, verified with the **TLC model checker**; transformation performed via the **fBPMN** tool suite.
- **5 case studies** of increasing complexity: Client-Supplier Model (CSM), Travel Agency Model (TAM), IoT Temperature Model (ITM), simplified Robot Controller Model (RCM), Internship Procedure Model (IPM) — complexity measured by node count and SF/MF edge ratio.
- **4 research questions:** RQ1 soundness verifiable? RQ2 safety verifiable? RQ3 verification efficient (time)? RQ4 is explored-transition-sequence length reasonable?
- **Environment:** 3.8GHz i7, 32GB RAM, JDK 17, TLA+ tools v1.6.0.

## 6. Key results

|Model|Nodes|SF/MF|Best-case (states/trans/depth)|Worst-case (states/trans/depth)|
|---|---|---|---|---|
|CSM|17|14/3|SYNC: 75/148/19|PLB: 95/176/26|
|TAM|20|18/5|SYNC: 229/548/17|PLAF: 562/941/40|
|ITM|37|29/8|SYNC: 441/944/46|PSAF: 721/1295/61|
|RCM|42|34/9|SYNC: 641/1132/56|PSAF: 817/1628/62|
|IPM|38|23/7|SYNC: 231/549/26|PSAF: 729/1295/60|

- **Verification time range: 3.27s (min) to 8.42s (max)** across all model/case-study combinations — fast enough to be practical.
- **State-space growth: 75→876 states, 141→1639 transitions, depth 22→63** — grows with case-study complexity but not explosively.
- **SYNC always produces the smallest transition system** (no buffering = simplest state space); **PLC/PSC (Causal) tend to produce the largest** (vector-clock bookkeeping adds state).
- **Cross-validation:** compared against existing verification frameworks (fBPMN, BProVe) on the same case studies — **zero discrepancies** in soundness/safety verdicts, supporting correctness of the new semantics.
- Not every model/case-study pair is sound or safe — Table 2 shows several ✗ marks (e.g., ITM under PLB/PSB fails both RQ1 and RQ2; TAM under PLAF/PSAF/PLC/PSC fails both), which is expected/useful — the framework is meant to _detect_ unsound/unsafe designs, not just confirm good ones.

## 7. Limitations (as stated by author)

- Verification time was measured only for the communication-model property, not more complex composite properties (mitigated by averaging 5 runs, but flagged as a validity threat).
- **Combinations of communication models** (e.g., different channels using different semantics within one collaboration) were not evaluated — could affect both verification time and validity of results.
- Communication models alone aren't sufficient to make BPMN fully "IoT-aware" — semantics for IoT-specific elements themselves (IoTActivity, IoTThrowEvent, IoTCatchEvent) are left as future work.

## 8. Why it matters / reusable insight

- Demonstrates a **complete formal-verification pipeline template**: formalize as typed graph → define configuration/transition semantics → translate to TLA+ → verify via TLC model checker → cross-validate against independent tools. This exact pipeline shape (formalize → translate to a checkable spec language → verify → empirically validate against baselines) is directly reusable for **SkillSpec** (guarded finite-state machines + V1–V4 verification algorithms) — worth comparing whether SkillSpec's verification algorithms could be expressed/cross-checked in TLA+ the same way, or whether the case-study/RQ structure here (soundness, safety, efficiency, state-space size) is a good template for SkillSpec's own evaluation section.
- The **vector-clock-based causal ordering** pattern (Listing 6) is a reusable primitive anywhere you need "preserve causal order without full global ordering" — could be relevant to MindPortalix's hash-chain audit logger or any multi-agent message-passing design where full synchronization is too strict but pure unordered delivery is too weak.
- Good evidence that **TLA+/TLC scales tractably** for realistic-sized BPMN collaborations (sub-10-second verification even at ~40 nodes) — useful data point if considering TLA+ as a verification backend for other formal-methods work (e.g., MindPortalix's governance layer or CI-gated governance tests).
- The **pull/push × ordering-semantics matrix** itself (Bag/Fifo/Causal/Sync) is a reusable taxonomy for reasoning about _any_ asynchronous message-passing system's delivery guarantees, not just BPMN/IoT — a handy mental checklist when designing agent-to-agent communication protocols (e.g., MCP integration in MindPortalix).

## 9. Citation

> Majid Babaei. 2024. Communication Semantics for IoT-aware Business Process Management Systems. In _2024 ACM/IEEE 6th International Workshop on Software Engineering Research & Practices for the Internet of Things (SERP4IoT '24)_, April 20, 2024, Lisbon, Portugal. ACM, New York, NY, USA, 8 pages. https://doi.org/10.1145/3643794.3648273