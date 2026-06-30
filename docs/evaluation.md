# Evaluation

We evaluate at **four levels** because pixel accuracy alone is misleading for road extraction — a model can have high IoU yet produce a fragmented, unroutable graph. Our north-star metrics are *topological* and *graph-level*.

---

## 1. Pixel-level (segmentation quality)

| Metric | Definition | Caveat |
|--------|-----------|--------|
| **IoU / Dice** | overlap of predicted vs GT road pixels | necessary, not sufficient; insensitive to breaks |
| **Precision / Recall / F1** | per-pixel | recall matters most (missed road = graph gap) |
| **Boundary F1** | F1 within a small distance of GT boundary | rewards crisp, continuous edges |

These confirm the model "sees" roads, but do **not** guarantee a usable graph — hence levels 2–4.

## 2. Topology-level (the primary signal)

| Metric | What it measures | Why it's our north star |
|--------|------------------|-------------------------|
| **APLS** (Average Path Length Similarity) | similarity of shortest-path lengths between GT graph and predicted graph over sampled node pairs | directly measures *routability* — penalizes breaks that pixel metrics ignore |
| **APLS-lite** | a faster APLS approximation on sampled O-D pairs | used as the **training checkpoint metric** |
| **TLTS / Too-Long-Too-Short** | fraction of routes whose length is within a tolerance of GT | complements APLS |
| **Connected-components ratio** | predicted #components vs GT (ideally 1) | fragmentation proxy; healing should drive this →1 |
| **Junction F1** | detected vs GT intersections within a radius | quality of Stage-1 junction head / Stage-2 nodes |

**APLS intuition:** sample pairs of points, compare the shortest-path distance in GT vs prediction. If a road is broken, predicted distances blow up (or become ∞) → APLS drops. This captures *exactly* the failure mode (fragmentation) that we built the whole pipeline to fix.

## 3. Healing efficacy (Stage 3 ablation)

Report the graph **before vs after** healing:

| Metric | Before heal | After heal | Goal |
|--------|-------------|-----------|------|
| # connected components | high | →1 | connectivity |
| APLS | lower | higher | routability gained |
| # healed edges added | — | small | minimal, plausible links |
| healed-edge precision* | — | high | links land on real roads |

\* *healed-edge precision*: of edges the healer added, what fraction coincide with a GT road (within tolerance). Demonstrates the **Connection Score** adds *correct* links, not just any links. Compare against the plain-MST baseline to quantify the value of confidence-guided scoring.

## 4. Graph / network-intelligence sanity

| Check | Expectation |
|-------|-------------|
| Betweenness distribution | heavy-tailed; arterials top-ranked (visual sanity vs known city arterials) |
| Bridges / articulation count | plausible for the city; located at real chokepoints (river crossings, single-access areas) |
| Communities (Louvain) | align with known districts/neighborhoods |
| Criticality vs ground truth events | flagged roads correlate with historically congested/flooded segments (if data available) |

## 5. Resilience validation (closes the loop)

The **targeted-vs-random resilience curve** is both a result *and* a validation of the criticality score:

```
remove top-k by criticality (targeted)  → efficiency should collapse fast
remove k at random                       → efficiency should degrade slowly
```
If targeted removal hurts far more than random, the criticality score is **identifying the truly critical roads** — a self-validating, demo-ready proof. Summarize with the **AUC of the resilience curve** (lower AUC under targeted attack = the network is fragile to losing exactly the roads we flagged).

## 6. System / performance metrics

| Metric | Target (demo scale) |
|--------|---------------------|
| Inference (full scene, tiled) | seconds–minutes on one GPU |
| Graph build (Stage 2–4) | < few seconds (CPU) |
| Analytics (Stage 5, cached after first) | first compute seconds; served instantly after |
| Simulation round-trip | **< 1 s** (cached graph) |
| Cache hit rate (repeat scenes/scenarios) | high → near-instant UX |

## 7. Ablations to present

| Ablation | Shows |
|----------|-------|
| Frozen vs fine-tuned encoder | frozen SAM is competitive at a fraction of train cost |
| With vs without junction head | junction-guided refinement improves node/topology metrics |
| Confidence-guided healing vs plain MST | higher APLS + healed-edge precision |
| With vs without occlusion augmentation | robustness gain on occluded test tiles |
| Hysteresis vs single threshold | fewer breaks, better APLS |

## 8. Reporting
A single `scripts/evaluate.py` produces a JSON + markdown report per scene: pixel metrics, APLS(-lite), components before/after heal, healed-edge precision, resilience-curve AUC, and timing. This report is the evidence slide for judging — **it leads with topology and resilience, not IoU.**

## 9. Honesty notes
- We report the **demo scene as held-out** (never trained on) — see [training.md](training.md).
- Healed edges are **flagged** in every output, so metrics and the dashboard never silently pass off inferred links as observed roads.
