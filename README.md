# Route Resilience

**Occlusion-Robust Road Extraction & Graph-Theoretic Criticality Analysis for Urban Mobility**

> Built for ISRO / NNRMS / MeitY-grade urban planning workflows. Designed to be realistically buildable by a small team in a ~30-hour national hackathon while presenting as a production-ready research prototype.

---

## 1. What this system does

Satellite imagery of cities is noisy: roads disappear under tree canopies, building shadows, vehicles, clouds, and seasonal change. Naive segmentation produces **fragmented road masks** that are useless for routing, disaster response, and GIS planning.

Route Resilience is an **end-to-end pipeline** that turns a raw satellite raster into an *actionable, queryable, stress-testable road network*:

1. **Occlusion-aware road + junction extraction** (deep model: `TopoSAM-RR`)
2. **Topology extraction** — skeleton → nodes → edges → raw graph
3. **Confidence-guided graph healing** — reconnect broken roads using geometry + model confidence
4. **Road attribute assignment** — length, direction, curvature, travel cost
5. **Network intelligence** — centrality, bridges, articulation points, communities, criticality scoring
6. **Stress testing** — node failure, road closure, flooding, congestion → resilience index
7. **Interactive planner dashboard** — overlays, heatmaps, simulations, rerouting

## 2. The core idea (why we win)

Most teams stop at "better segmentation IoU." We treat segmentation as **stage 1 of 7** and make the *graph* the first-class product. Three differentiators:

| Differentiator | What it means | Why judges care |
|----------------|---------------|-----------------|
| **Occlusion-robust extraction** | A junction-aware decoder + confidence maps, not just a binary mask | Roads under canopy/shadow are recovered, not dropped |
| **Confidence-guided healing** | Broken topology is reconnected using a learned/geometric *Connection Score*, not a blind MST | The graph is actually *connected and routable* |
| **Resilience-as-a-product** | Criticality scoring + failure simulation on the reconstructed graph | Directly answers "which roads must we protect?" — the planner's real question |

## 3. High-level architecture

```
                 ┌────────────────────────────────────────────────────────┐
 Satellite       │                   ML INFERENCE PLANE                    │
  raster   ─────▶│  Tiling → Normalize → TopoSAM-RR → Road Prob + Junction │
                 └───────────────────────────┬────────────────────────────┘
                                             ▼
                 ┌────────────────────────────────────────────────────────┐
                 │                 GRAPH / TOPOLOGY PLANE                  │
                 │  Skeleton → Nodes/Edges → Healing → Attributed Graph →  │
                 │  Centrality / Bridges / Articulation / Communities      │
                 └───────────────────────────┬────────────────────────────┘
                                             ▼
                 ┌────────────────────────────────────────────────────────┐
                 │             SIMULATION / ANALYTICS PLANE               │
                 │  Scenario apply → Resilience metrics → Reroute / Delay  │
                 └───────────────────────────┬────────────────────────────┘
                                             ▼
                 ┌────────────────────────────────────────────────────────┐
                 │      API PLANE (FastAPI)  ·  Cache (Redis)  ·  Jobs     │
                 └───────────────────────────┬────────────────────────────┘
                                             ▼
                 ┌────────────────────────────────────────────────────────┐
                 │   FRONTEND (Next.js + MapLibre/deck.gl) — Planner UI    │
                 └────────────────────────────────────────────────────────┘
```

See [docs/architecture.md](docs/architecture.md) for the full design.

## 4. Documentation suite

| Doc | Contents |
|-----|----------|
| [architecture.md](docs/architecture.md) | System architecture, planes, data flow, scalability |
| [model.md](docs/model.md) | TopoSAM-RR: encoder/decoder/heads/tensors/losses |
| [pipeline.md](docs/pipeline.md) | Stage 1–2: segmentation → topology extraction |
| [graph_engine.md](docs/graph_engine.md) | Stage 3–5: healing, attributes, network intelligence |
| [simulation.md](docs/simulation.md) | Stage 6: stress testing & resilience |
| [backend.md](docs/backend.md) | FastAPI services, jobs, caching, DI, config |
| [frontend.md](docs/frontend.md) | Next.js dashboard, map rendering, state |
| [api.md](docs/api.md) | REST endpoints & contracts |
| [deployment.md](docs/deployment.md) | Docker, K8s, object storage, scaling |
| [tech_stack.md](docs/tech_stack.md) | Every technology choice + justification |
| [math.md](docs/math.md) | Math + intuition for every algorithm |
| [training.md](docs/training.md) | Datasets, labels, schedule, hackathon fallbacks |
| [evaluation.md](docs/evaluation.md) | Metrics: pixel, topology, graph, resilience |
| [future_scope.md](docs/future_scope.md) | Roadmap & extensions |

## 5. Repository layout

```
route-resilience/
├── backend/        FastAPI app: API plane, services, workers
├── ml/             TopoSAM-RR model, training engine, inference
├── graph/          Topology extraction, healing, attributes, analytics
├── simulation/     Scenario engine + resilience metrics
├── frontend/       Next.js + MapLibre/deck.gl planner dashboard
├── deployment/     Dockerfiles, compose, k8s manifests
├── configs/        YAML configs (model, pipeline, app)
├── scripts/        CLI utilities (tile, infer, build-graph, demo-seed)
├── tests/          unit / integration / e2e
├── data/           raw / processed / tiles (gitignored)
├── artifacts/      model checkpoints (gitignored / object storage)
└── docs/           the documentation suite above
```

## 6. Quickstart (target)

```bash
# 1. Spin up the stack
docker compose -f deployment/compose/docker-compose.yml up --build

# 2. Seed a demo city (pre-baked tiles + checkpoint)
python scripts/demo_seed.py --city demo_city

# 3. Open the dashboard
open http://localhost:3000
```

## 7. Hackathon execution strategy (30h)

The system is designed so the **demo never depends on training finishing**. See [docs/training.md](docs/training.md) for the fallback ladder (pretrained SAM weights → fine-tune head only → pre-baked inference cache). The graph/simulation/dashboard planes run on cached inference outputs, so they can be built and demoed in parallel with model work.

## 8. Status

This repository currently contains the **complete engineering design + scaffold**. Implementation tickets are tracked per stage in the docs.
