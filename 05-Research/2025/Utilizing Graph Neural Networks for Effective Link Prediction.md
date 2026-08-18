
**Type:** Full paper (12 pages) | **Venue:** ICPE '25 — 16th ACM/SPEC Int'l Conf. on Performance Engineering (Toronto, May 5–9, 2025) **Authors:** Ghazal Khodabandeh (Brock), Alireza Ezaz (Brock), Majid Babaei (McGill), Naser Ezzati-Jivan (Brock) **DOI:** 10.1145/3676151.3719362 **Code:** https://github.com/ghazalkhb/Link_Prediction **Tags:** #AI4SE #GNN #GAT #link-prediction #microservices #distributed-systems #ICPE2025

---

## 1. One-line summary

Uses a **Graph Attention Network (GAT)** with **temporal windowing** and **degree-weighted negative sampling** to predict future service-to-service calls in microservice Call Graphs — enabling proactive monitoring instead of reactive incident response.

## 2. Problem it solves

- Microservice Call Graphs are **high-frequency, time-sensitive, and heavy-tailed** (a few hub services, many sparse ones) — unlike social networks where traditional link prediction was developed.
- Existing link-prediction methods (similarity-based, matrix factorization, plain GNNs) either ignore temporal dynamics or ignore structural centrality.
- Goal: forecast which service-to-service interactions will occur next, so operators can proactively address bottlenecks/failures before cascading.

## 3. Core contributions

1. **Temporal segmentation** — fixed time windows (not sliding) turn the trace into a sequence of independent graphs, letting the model learn short- and long-term dependency patterns without added computational complexity.
2. **Advanced (degree-weighted) negative sampling** — counters extreme class imbalance (positives = observed calls; everything else looks like a "negative") by biasing sampling toward high-degree (central/hub) nodes rather than uniform random pairs.
3. **Extensive real-world evaluation** on Alibaba's 2022 microservice cluster trace (~20K microservices, 13 days) validating practicality for proactive monitoring.

## 4. Method pipeline (Fig. 1 in paper)

`DB → Data Cleaning (DCP) → Node Mapping Policy (NMP) → Time Window Adjustment (TWA) → Graph Construction (per window) → Sampling Analysis (choose ANS/SNS/NS) → Train GAT → Test + Visualize`

**Data model:** trace `T = {(s_i, d_i, t_i, A_i)}` — caller, callee, Unix timestamp, attribute vector (service name, RPC type, latency — attributes not used in modeling itself, only timestamp).

**Preprocessing steps:**

- Data Cleaning Phase (DCP): drop irrelevant/incomplete records, sort by timestamp.
- Node Mapping Policy (NMP): map raw service IDs → standardized integer node IDs.
- Time Window Adjustment (TWA): fixed windows (e.g., 0–100ms, 100–200ms…) → each window is an independent graph.
- Node features: **identity matrix only** (no engineered features) — deliberately simple, relies on structure + attention rather than hand-crafted features.

**Negative sampling — 3 strategies evaluated, chosen per dataset balance:**

- No Sampling (|P| ≈ |N|)
- Simple Negative Sampling (uniform random non-edges, for moderate imbalance)
- **Advanced Negative Sampling (chosen for this dataset):** `p(v) = d_v^α / Σ_u d_u^α` — sample probability weighted by node degree^α (α tuned; higher α → favors hub nodes as negative-sample endpoints), explicitly excluding real edges.

**Model — GAT core:**

- 2 graph attention layers: layer 1 = multi-head (2 heads) attention + ELU activation; layer 2 = single-head consolidation.
- Attention coefficient: `α_ij = softmax_j(LeakyReLU(aᵀ[Wh_i‖Wh_j]))`
- Node update: `h'_i = σ(Σ_j∈N(i) α_ij · W h_j)`, multi-head version concatenates K heads.
- Link probability: dot product of node embeddings → sigmoid: `p_ij = σ(h_iᵀ·h_j)`; threshold τ tuned for precision/recall balance.
- Loss: binary cross-entropy over positive + sampled negative edges.
- Temporal info embedded via timestamp features on nodes/edges (not a separate temporal-attention module in the final model, despite TGAT/TGN being discussed as related work).

## 5. Experimental setup

- **Dataset:** Alibaba 2022 Cluster Trace (microservices), ~20,000 microservices, 13-day trace.
- **Split:** train on timestamps 0–7000ms (589,540 records), test on 7000–10000ms (250,117 records) — strict temporal split, no leakage.
- **Hardware:** 51GB RAM, 225.8GB disk, ~62.7% idle CPU — modest, CI-feasible setup.
- **Metrics:** AUC, Accuracy, Precision, Recall, F1 (+ interpretability: PR curve, ROC curve, confusion matrix, attention heatmaps).

## 6. Key results

|Method|AUC|Acc.|Prec.|Rec.|F1|
|---|---|---|---|---|---|
|NodeSim (structural, random-walk)|0.50|0.60|0.40|0.17|0.18|
|Adjusted NodeSim|0.62|0.69|0.39|0.20|0.10|
|LSTM (temporal only)|0.76|0.73|0.51|0.54|0.60|
|Simple GNN (structural only)|0.94|0.69|0.62|0.97|0.76|
|Simple Temporal GNN|0.93|0.72|0.65|0.96|0.77|
|**Our Approach (GAT + temporal windows + ANS)**|0.89|**0.91**|**0.89**|0.96|**0.92**|

**Takeaway:** Our approach isn't the single highest on any one raw metric (Simple GNN edges it on AUC/recall) but wins decisively on the metrics that matter for actionable predictions — **accuracy, precision, and F1** — meaning far fewer false positives/negatives in practice. Confusion matrices confirm it has the best balance of true positives/negatives among all baselines (Fig. 2).

**Qualitative findings:**

- Attention heatmaps stabilize by ~epoch 199, showing convergence to consistent, meaningful edge importances.
- PR curves vary by time window (Window 0 smooth decline vs. Window 12 sharper early drop = occasional overconfidence) but stay robust overall.
- ROC curve (Window 21) shows AUC approaching 1, consistent across windows.

## 7. Limitations / threats to validity (as stated by authors)

- Evaluated only on binary classification metrics (AUC/P/R/F1), not ranking metrics (MRR, Hits@K) — future work if ranking becomes the goal.
- Single dataset (Alibaba trace), single 10,000ms window duration — generalizability to other durations/datasets untested.
- GNN computational cost > simpler models (LSTM, NodeSim) — scalability/lightweight-architecture tradeoffs flagged as future work.
- No node/edge attributes beyond identity + timestamp used (deliberately simple) — richer features (latency, RPC type, service load) could improve results further.

## 8. Why it matters / reusable insight

- Reinforces a broader pattern across your work (SIKG, MindPortalix): **combining structural graph signals with temporal/behavioral history consistently beats either alone.**
- The **degree-weighted advanced negative sampling** trick (`p(v) ∝ d_v^α`) is a clean, reusable idea for any sparse-positive graph-learning task with severe class imbalance — could apply to SIKG's test-impact graph or MindPortalix's dependency graph if training a learned component there.
- Fixed time-window graph segmentation (vs. sliding windows) is a simple, low-overhead way to add temporal awareness to any static-graph pipeline — worth considering wherever "snapshot" graphs are built.
- Good venue-fit precedent: ICPE (performance engineering) rewards this kind of "predict the interaction/impact ahead of time to enable proactive ops" framing — same narrative arc used successfully in SIKG (ICSE) and could work for future governance/monitoring stories (MindPortalix, ARMOR).

## 9. Citation

> Ghazal Khodabandeh, Alireza Ezaz, Majid Babaei, and Naser Ezzati-Jivan. 2025. Utilizing Graph Neural Networks for Effective Link Prediction in Microservice Architectures. In _Proceedings of the 16th ACM/SPEC International Conference on Performance Engineering (ICPE '25)_, May 5–9, 2025, Toronto, ON, Canada. ACM, New York, NY, USA, 12 pages. https://doi.org/10.1145/3676151.3719362