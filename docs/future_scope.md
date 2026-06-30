# Future Scope

Route Resilience is architected so each plane can deepen independently (artifact contracts isolate change). Below: what we'd build next, ordered by impact-to-effort, and why the current design already accommodates it.

---

## 1. Model & perception
- **SAM2 hierarchical encoder** — native multi-scale features remove the ViT-FPN projection trick; better thin-road recovery. (Drop-in: same decoder/heads.)
- **Temporal / multi-date fusion** — combine seasonal images (leaf-on/leaf-off) so canopy occlusion in one date is filled from another. Directly attacks the core problem.
- **Multi-modal inputs** — fuse SAR (cloud-penetrating) and DEM (slope/road plausibility) channels; SAR makes the pipeline weather-robust.
- **Road *class* head** — predict highway/arterial/local → better travel-cost priors and routing realism.
- **Uncertainty estimation** — MC-dropout/ensembles → calibrated `seg_confidence`, making healing scores even more trustworthy.

## 2. Topology & healing
- **Learned healing (GNN link prediction)** — replace the hand-tuned Connection Score weights with a small GNN trained on (gap → real-link?) examples. The scoring engine is already a clean module to swap.
- **Lane/direction extraction** — one-way streets, turn restrictions → directed graph for realistic routing.
- **Vectorization to OSM schema** — export healed graph as OSM-compatible data for direct GIS/routing-engine ingestion.

## 3. Analytics & simulation
- **Demand-weighted criticality** — weight betweenness by real O-D demand (population, POIs) instead of uniform pairs → "critical for *actual* trips."
- **Cascading failure model** — congestion redistributes load and overloads neighbors (traffic-assignment) → realistic disaster propagation, not just static removal.
- **Multi-modal resilience** — add rail/bus layers; evaluate whether transit cushions road failures.
- **Optimization** — given a budget, recommend *which* bridges/roads to reinforce to maximize resilience gain (greedy / submodular over the resilience curve).

## 4. Data & scale
- **Streaming tile pipeline** — process a city as tiles stream in; progressive dashboard rendering (the queue/worker design already supports fan-out).
- **Continental scale** — partition by admin region; per-region graphs stitched at borders; igraph/rustworkx for centrality at scale.
- **Change detection** — re-run on new imagery, diff graphs → "new road built / road washed out" alerts for planners.

## 5. Product & UX
- **Scenario library & sharing** — save/share named scenarios ("2023 flood extent") with permalinks.
- **Report export** — one-click PDF/GeoPackage for a planning report (the `evaluate.py` report is the seed).
- **Collaborative planning** — multiple planners annotate the same scene (the stateless API + scene-scoped keys already isolate sessions).
- **What-if road *addition*** — let planners *draw a proposed new road* and see the resilience gain (inverse of failure simulation — same engine).

## 6. MLOps & production
- **Active learning loop** — planners correct mis-extracted roads in the UI → corrections become training labels → model improves over deployments.
- **Model registry + canary** — version checkpoints, canary new models on held-out scenes before promotion (registry already keyed by tag).
- **Full observability** — Prometheus/Grafana + tracing across API→worker→artifact (correlation ids already threaded).

## 7. Domain extensions (the ISRO/NNRMS/MeitY pitch)
- **Disaster response mode** — ingest near-real-time imagery post-flood/earthquake, extract passable roads, route relief vehicles around damage.
- **Rural connectivity (PMGSY-style)** — find communities with single-bridge access (articulation analysis) → prioritize infrastructure investment.
- **Urban planning integration** — feed criticality into land-use/transport models; expose as a GIS plugin (QGIS) consuming the GeoPackage exports.

---

## Why the architecture is ready for all this
Every item above changes **one plane** behind a **stable artifact contract**:
- a better model still emits `road_prob.tif` / `junction.tif`;
- learned healing still emits a connected `graph.json`;
- new scenarios are just more graph transforms;
- new analytics are additional cached layers.

So the system can grow from a hackathon prototype to a production geospatial platform **without re-plumbing** — which is exactly the scalability claim the design was built to support. See [architecture.md](architecture.md) and [deployment.md](deployment.md).
