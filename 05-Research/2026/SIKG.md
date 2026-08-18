# SIKG — Semantic Impact Knowledge Graph for Adaptive Test Selection & Prioritization

**Type:** Poster paper | **Venue:** ICSE-Companion '26 (Rio de Janeiro, Apr 12–18, 2026) **Authors:** Majid Babaei (UFV), Naser Ezzati-Jivan (Brock), Lili Wei (McGill) **DOI:** 10.1145/3774748.3795655 **Code:** https://github.com/ai4se4ai-lab/SIKG **Tags:** #AI4SE #regression-testing #test-selection #test-prioritization #knowledge-graph #reinforcement-learning #ICSE2026

---

## 1. One-line summary

SIKG replaces coverage-only regression test selection/prioritization with a **semantic, knowledge-graph-driven** approach that understands _why_ code changed, propagates that impact through a weighted graph, and improves itself over time via reinforcement learning.

## 2. Problem it solves

Traditional Regression Test Selection (RTS: Ekstazi) and Test Case Prioritization (TCP) methods:

- Rely on structural code coverage only (file/method level).
- Treat all changes uniformly — no notion of _change intent_ (bug fix vs. refactor vs. feature).
- RL-based TCP approaches learn from execution history but still lack semantic understanding.

## 3. Core idea — 5-phase framework

1. **Knowledge Graph Construction** Graph G = (V, E, L, W): nodes = code elements (classes/methods) + test cases. Edge weights combine static coupling metrics with 3 empirical/historical signals:
    
    - Co-change frequency (Jaccard coefficient on commit history)
    - Empirical impact strength (how often a test fails when specific code changes)
    - Fault correlation (test's historical fault-detection effectiveness in a code region)
2. **Semantic Change Analysis** On each commit: parses Git diff via AST + extracts textual signals from commit messages → multi-class classifier labels change type (BUG_FIX, FEATURE_ADDITION, REFACTORING, etc.) Formula: `impact_initial(τ, node) = base(τ) × centrality(node)` → produces semantic change tuple `s = (element, type, impact)`
    
3. **Impact Propagation** BFS traversal from changed nodes, bounded by max depth `d_max`. Score at each node driven by: edge weight, distance-based attenuation, semantic relevance multiplier. Threshold `θ_min` prunes low-impact nodes.
    
4. **Test Selection & Prioritization**
    
    - Selection: tests with `impact(t) ≥ θ_selection` → subset T′
    - Prioritization score: `objScore(t) = α·impact(t) + β·effectiveness(t) − γ·execTime(t)`
5. **Reinforcement Learning (self-improvement loop)** MDP framing: states = code changes + test history, actions = selection decisions, rewards = outcome accuracy. Prediction error δ from actual test outcome (fail=1.0, pass=0.0, inconclusive=0.5) drives continual re-calibration of impact predictions.
    

## 4. Evaluation setup

- 7 open-source projects: Django, Pandas, NumPy, Scikit-learn, Requests, Flask, Pytest
- 2.3M LOC, 99,845 tests, spanning web/data/scientific/ML/HTTP/testing-framework domains
- Baselines: Random, Coverage-RTS, History-TCP, SIKG-NoEnrich (ablation w/o empirical weight enrichment)

## 5. Key results (by RQ)

|RQ|Finding|
|---|---|
|RQ1 – KG effectiveness|SIKG: **P 0.887 / R 0.944 / F1 0.915** — beats SIKG-NoEnrich (0.734P/0.808F1, +20.8%P/+13.2%F1), Coverage-RTS (0.645P/0.732F1), History-TCP (0.587P/0.686F1)|
|RQ2 – Semantic analysis|Change-type classification accuracy: BUG_FIX 93.7%, DEPENDENCY_UPDATE 94.3%, REFACTORING_SIGNATURE 91.8%, FEATURE_ADDITION 88.2%, PERFORMANCE_OPT 86.1%, REFACTORING_LOGIC 82.4%. Optimal `d_max=3` → 87.8%P/91.3%R. At 40% suite reduction: 85.8% fault detection (vs 67.8% Coverage-RTS, 60.1% History-TCP)|
|RQ3 – RL adaptation|+16.7% accuracy over 500 consecutive changes. +26.3% early fault detection vs static version. Domain learning curves: web frameworks +18.2% (after 200 changes), data science +15.8% (gradual), testing frameworks +12.4% (steady). Overhead: semantic analysis 38ms/file, propagation 95ms/change, selection 29ms/impact-set, RL update 12ms/result|
|RQ4 – Cross-domain scalability|Consistent across domains (F1 std dev = 1.0%). Avg test-suite reduction **74.4%**, fault detection **78.9%** at 60% reduction. Best: Web Framework (F1 0.929, FDR@60% 81.4%). Weakest-but-strong: Testing Framework (F1 0.902, FDR@60% 75.1%, due to self-referential dependencies)|

**Headline numbers to remember:** 74.4% avg test suite reduction · 78.2–78.9% fault detection at 60% reduction · 162ms avg total analysis time (CI-pipeline practical).

## 6. Why it matters / reusable insight

- Demonstrates that **change-intent-aware weighting** (semantic type + empirical history) meaningfully beats purely structural/coverage-based RTS/TCP.
- The **RL feedback loop** (predicted vs. actual test outcome → recalibrate impact scores) is a reusable pattern for any "prediction system operating in CI" — could generalize beyond test selection (e.g., flaky test detection, build-failure prediction).
- Five-phase pipeline (KG construction → semantic analysis → propagation → selection/prioritization → RL) is a clean template for other AI4SE graph-based systems.

## 7. Possible follow-ups / connections to other projects

- Architecturally related to **MindPortalix**'s governance/graph work — both use graph-based propagation + audit/feedback loops; worth comparing edge-weighting schemes.
- The impact-propagation + threshold-pruning design (θ_min, d_max) could inform **SkillSpec**'s verification-algorithm framing (bounded traversal with formal guarantees).
- Reinforcement-learning-for-calibration pattern could be relevant to Mitacs/Enya Learning's adaptive-learning IU#1 (LangGraph configuration space).

## 8. Citation

> Majid Babaei, Naser Ezzati-Jivan, and Lili Wei. 2026. SIKG: A Semantic Impact Knowledge Graph Approach for Adaptive Test Selection and Prioritization. In _2026 IEEE/ACM 48th International Conference on Software Engineering (ICSE-Companion '26)_, April 12–18, 2026, Rio de Janeiro, Brazil. ACM, New York, NY, USA, 2 pages. https://doi.org/10.1145/3774748.3795655