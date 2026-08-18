# TAAF: Trace Abstraction and Analysis Framework (Synergizing Knowledge Graphs and LLMs)

**Full title:** TAAF: A Trace Abstraction and Analysis Framework Synergizing Knowledge Graphs and LLMs **Authors:** Alireza Ezaz, Ghazal Khodabandeh (Brock University); Majid Babaei (McGill University); Naser Ezzati-Jivan (Brock University) **Venue:** Accepted for publication at **ICSE 2026** **DOI:** 10.1145/3744916.3787832 **arXiv:** 2601.02632v1 [cs.SE] **Code/data:** https://anonymous.4open.science/r/TAAF-LLM-KG-State-System--60EC/README.md

---

## TL;DR

TAAF turns massive, low-level kernel/system execution traces (millions–billions of events) into natural-language answers by **not** feeding raw traces to an LLM. Instead it builds a pipeline: raw trace → **time-indexed State System** (via Trace Compass-style abstraction) → **query-specific knowledge graph** → **LLM reasoning over the graph**. This structured grounding improves QA accuracy by up to **31.2%** over giving the LLM raw/numeric trace output, especially for multi-hop and causal questions. The paper also introduces **TraceQA-100**, the first public benchmark for LLM reasoning over real kernel traces.

---

## Problem It Solves

Trace analysis (from tools like LTTng, Trace Compass, Zipkin, Jaeger) requires deep OS/hardware expertise and today mostly means either using rigid predefined dashboards or writing custom low-level scripts — slow (15–45 min per investigation per prior work) and inaccessible to non-experts. Feeding raw traces directly to an LLM doesn't work because:

1. **Traces are massive** — tens/hundreds of millions of events, far beyond any LLM context window.
2. **Traces are multidimensional** — threads, CPUs, files, sockets all interacting over time.
3. **Traces lack semantics** — raw numeric fields give the LLM no structure to reason over, causing hallucination.

## Core Idea: 3-Layer Pipeline (T → S → G → A)

```
Raw Trace (T) --φ1--> State System (S) --φ2--> Query-specific Knowledge Graph (Gq) --φ3--> Answer (Aq)
```

**Layer 1 — Trace → State System (φ1):** Converts the raw event stream into a **time-indexed structured abstraction**, reusing the "State System" concept from Trace Compass (Montplaisir et al. 2013): two synchronized trees —

- an **attribute tree** of stable integer IDs ("quarks") for hierarchical paths (e.g. `/CPU/2/Thread/5130`)
- a **history tree** recording value-transition intervals `⟨t_start, t_end, quark, value⟩`

This supports `query(q, t) → v` and `query(q, [t1,t2]) → {vi}` in roughly logarithmic time relative to trace size — no full trace scans needed.

**Layer 2 — State System → Query-specific Knowledge Graph (φ2):** Given a natural-language query, TAAF pulls only the relevant time interval/entities from the State System and builds a small, **on-demand knowledge graph** `Gq = (Vq, Eq, μq, τq)`:

- `Vq` = typed entity nodes (Thread, CPU, etc.)
- `Eq` = labeled directed edges (relations like `executes_on`, `reads_from`, `holds_lock`)
- `μq` = edge weights (duration, frequency, etc.)
- `τq` = temporal scope per node/edge

Crucially the graph is **query-scoped, not global** — avoids overgeneralized/unwieldy graphs and keeps things interpretable.

**Layer 3 — Graph → Answer via LLM (φ3):** The graph is serialized as JSON along with a **schema prompt** (explicitly defining node/edge types and attributes) and the user's question, then passed to an LLM. This constrains the model to reason over a well-scoped, explicit context rather than open-ended inference — reducing hallucination and improving explainability/auditability.

## TraceQA-100 Benchmark

First public benchmark pairing large-scale kernel traces with expert-verified QA:

- Built from **LTTng traces of SciMark 2.0** (Java benchmark), ~34M kernel events per run.
- Slices taken at 3 temporal locations (start/mid/end) × 3 window lengths (1s/10s/100s) = 9 trace segments.
- **100 questions**: 40 explanatory, 30 multiple-choice, 30 true/false; 50/50 split single-hop vs multi-hop.
- Each question instantiated across all 9 slices → **900 unique question-trace pairs**.
- Ground truth authored via a 4-step double-blind process (2 experts draft, peer review, 3rd expert produces reference answers via hand-written Trace Compass scripts, disagreements adjudicated).

## Evaluation Setup

- **3 trace representations compared:** Events-only (raw, exceeds context — qualitative only), Baseline (raw State System numeric output), TAAF (query-specific KG).
- **Models (main grid):** GPT-4.1 nano, GPT-4o, o4-mini (reasoning).
- **Cross-family probe:** Gemini 2.5 Pro/Flash, Claude Opus 4.1/Haiku 4.5 — to check vendor-independence.
- **Scoring:** 3-point scale (0 = wrong, 0.5 = partial, 1 = correct), each question sampled 3× for stochasticity.
- **Metrics:** weighted **Accuracy** and an entropy-based **Consistency** score.
- Total dataset: **7,500 labeled responses** (5,400 from Phase 1 full-grid + 2,100 from Phase 2 focused ablations).
- Efficiency: full pipeline (indexing + KG build + inference) on a 100s trace completes in **under 40 seconds** — feasible for interactive use.

## Key Results by Research Question

|RQ|Question|Key Finding|
|---|---|---|
|**RQ1**|Accuracy by query type / hop count?|True/False single-hop highest (90.99%). Explanatory multi-hop hardest (37.22% baseline). KG helps in **every** cell; biggest gain on True/False multi-hop (+30.12pp → 79.63%).|
|**RQ2**|Effect of time-window length?|Accuracy falls as window widens (1s→10s→100s) for all models, but TAAF degrades **2–3× slower** than baseline. o4-mini stays >90% even at 100s.|
|**RQ3**|Does KG grounding actually help?|Yes, consistently. Mean gain **+21.5%** across all 9 model-interval combos; range **+8.67% to +31.17%**. KG mainly converts wrong/partial answers into fully correct ones (not just more "0.5" scores).|
|**RQ4**|Variance across LLM backends?|o4-mini + TAAF tops every interval (95.5% at 1s, 90.17% at 100s). GPT-4o ~15pp behind. GPT-4.1 nano stays in 50–60% band even with KG. Cross-family probe (Gemini 2.5 Pro 95.83%, Gemini 2.5 Flash 89.50%, Claude Opus 4.1 87.67%, Claude Haiku 4.5 73.67%) shows the same reasoning-flagship-vs-compact-model pattern — confirms gains come from **graph grounding**, not a specific vendor.|
|**RQ5**|Does passing the graph schema (not just raw triples) help?|Yes — **+8.1pp** absolute accuracy (71.7% → 79.8%) just from adding the lightweight JSON schema describing node/edge types.|
|**RQ6**|Does temporal window placement (start/mid/end) matter?|Small effect: accuracy climbs 77.0% (start) → 79.3% (mid) → 81.5% (end), ~4.5pp spread. Late-trace windows have steadier execution patterns; TAAF is fairly robust to window placement.|
|**RQ7**|Effect of sampling temperature?|Best accuracy+consistency trade-off at **temperature 0.1–0.3**; accuracy/consistency degrade above 0.7. Authors used 0.5 as their overall default.|

## Related Work Positioning

- **Trace abstraction:** builds directly on Trace Compass's **State System** (Montplaisir et al. 2013) for scalable time-indexed querying — but the State System alone lacks semantic structure/flexible querying, which is the gap TAAF's KG layer fills.
- **Other trace abstraction work** (Pirzadeh's Gestalt-based phase segmentation, Feng et al.'s "Sage" hierarchical abstraction, Hamou-Lhadj's summarization/Utilityhood metric, Cornelissen et al.'s trace-reduction comparison) — all reduce trace size but don't add semantic/graph structure for LLM consumption.
- **KGs for system modeling:** draws on Liang et al.'s taxonomy (static/temporal/multimodal KGs); notes few systems build KGs _dynamically from execution traces_ — this is TAAF's specific contribution.
- **LLM–KG integration:** situates itself among RAG-style grounding (vs. embedding KG structure into model weights via adapters/LoRA/QA-GNN) — TAAF is essentially a **query-scoped RAG pattern applied to system traces**, closer to "compressed history trees" work than to fine-tuning-based approaches.

## Threats to Validity (as stated by authors)

- **Construct:** only 100 hand-crafted questions from a single benchmark (SciMark 2.0); coarse 3-level scoring rubric may miss small numeric errors.
- **Internal:** author-labeling introduces potential bias; large graphs/traces sometimes require truncating edge attributes to fit context.
- **External:** all traces are Linux/SciMark 2.0 — generalization to other kernels/workloads (eBPF, HPC, mobile) untested; API-based LLMs may silently change over time, hurting reproducibility. Global/unbounded queries (not window-scoped) are explicitly **out of scope** — KGs would become too large.
- **LLM reasoning limits:** even with correct KG context, models sometimes fail multi-step aggregation questions (e.g., "which CPU has the lowest difference between its highest and lowest thread times") — arithmetic/step-skipping errors observed.
- **Conclusion:** only 3 samples per question; baseline comparison is only against raw State System, not other intermediate designs (e.g., static graphs, summaries).

## Practical Takeaways (for reuse)

- **The T → S → G → A pipeline pattern is broadly reusable** beyond kernel traces: any domain with (a) huge low-level event logs, (b) a need for time-scoped natural-language QA, and (c) an existing time-indexed abstraction (like Trace Compass's State System) could adopt this same "index → query-scope into a small KG → let the LLM reason over the KG" recipe instead of naive RAG-over-raw-logs.
- **Schema-in-prompt is a cheap, high-leverage trick** — just describing node/edge types explicitly in the prompt (not fine-tuning, not more retrieval) bought +8pp accuracy. Worth trying whenever passing structured/graph data to an LLM.
- **Query-scoped graphs beat global graphs** for grounding — deliberately keeping the KG small and tailored to the specific question avoids both context bloat and "overgeneralization" noise. Useful design principle for any RAG-over-structured-data system.
- **Shorter time windows = safer answers.** If building a similar tool, default to narrow windows and let the user (or an agent) expand only if needed — accuracy drops noticeably past ~10s windows, especially for smaller/cheaper models.
- **Low temperature (0.1–0.3) is the sweet spot** for factual/structured QA tasks like this — a good default recommendation for similar grounded-QA systems.
- **Reasoning-tuned models (o4-mini, Gemini 2.5 Pro) benefit disproportionately more** from graph grounding than compact fast models (GPT-4.1 nano, Claude Haiku) — if cost allows, pairing a reasoning-tier model with structured grounding gives the best ROI; compact models may not be worth it for complex multi-hop trace questions.
- **TraceQA-100 itself is a reusable artifact** — worth checking if it becomes a standard benchmark for future trace-analysis / systems-QA tool evaluation.

## Citation

Ezaz, A., Khodabandeh, G., Babaei, M., Ezzati-Jivan, N. (2026). TAAF: A Trace Abstraction and Analysis Framework Synergizing Knowledge Graphs and LLMs. _Proceedings of ICSE 2026_. https://doi.org/10.1145/3744916.3787832