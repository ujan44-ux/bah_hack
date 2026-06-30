# System Architecture

This document is the architectural source of truth for **Route Resilience**. It defines the planes, the modules in each plane, the objects that flow between them, and the scalability strategy. Stage-internal designs live in [pipeline.md](pipeline.md), [graph_engine.md](graph_engine.md), [simulation.md](simulation.md), and [model.md](model.md).

---

## 1. Design philosophy

> The road **graph** — not the segmentation mask — is the product.

Everything is organized so that each plane produces a **durable, cacheable artifact** that the next plane consumes. This gives us three properties that win hackathons:

1. **Parallel buildability** — once the ML plane emits a cached `road_prob.tif`, the graph/sim/UI teams build against it without a GPU.
2. **Demo robustness** — every artifact is checkpointed to object storage; the live demo replays cached artifacts, so nothing depends on a model finishing training.
3. **Scalability narrative** — each plane is independently horizontally scalable because the contract between planes is *files + a job id*, not in-memory state.

---

## 2. The five planes

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. ML INFERENCE PLANE      (ml/)        GPU, batch, async             │
│    raster → tiles → tensor → TopoSAM-RR → road_prob + junction_heat   │
├─────────────────────────────────────────────────────────────────────┤
│ 2. GRAPH / TOPOLOGY PLANE  (graph/)     CPU, deterministic            │
│    masks → skeleton → nodes/edges → healed graph → attributed graph   │
│    → centrality / bridges / articulation / communities / criticality │
├─────────────────────────────────────────────────────────────────────┤
│ 3. SIMULATION PLANE        (simulation/) CPU, interactive             │
│    base graph + scenario → perturbed graph → resilience metrics       │
├─────────────────────────────────────────────────────────────────────┤
│ 4. API / ORCHESTRATION     (backend/)   FastAPI + Redis + Celery      │
│    jobs, caching, artifact registry, REST contracts, live updates     │
├─────────────────────────────────────────────────────────────────────┤
│ 5. PRESENTATION PLANE      (frontend/)  Next.js + MapLibre + deck.gl  │
│    overlays, criticality heatmap, simulation controls, rerouting      │
└─────────────────────────────────────────────────────────────────────┘
```

### Why planes, not microservices?
A **modular monolith** for the backend (single FastAPI process) with the ML/graph/sim code as importable packages. Microservice boundaries are drawn *logically* (the planes) but deployed as 2 processes (API + worker) for the hackathon. The K8s manifests show how each plane splits into its own service later — the seams are already there because each plane talks via artifacts + job ids.

---

## 3. End-to-end data flow (every object named)

```
Satellite GeoTIFF (raster, CRS, N bands)
        │  rasterio windowed read
        ▼
Raster Tiles            tile_{z}_{x}_{y}.tif         (overlapping, 512²)
        │  normalize + to-tensor
        ▼
Normalized Tensor       [B, 3, 512, 512] float32
        │  TopoSAM-RR forward
        ▼
Feature Maps            encoder {1/4,1/8,1/16}, decoder pyramid
        │  heads
        ▼
Road Probability Map    road_prob.tif   [H, W] float32 ∈ [0,1]
Junction Heatmap        junction.tif    [H, W] float32 ∈ [0,1]
        │  threshold + morphology
        ▼
Binary Road Mask        road_mask.tif   [H, W] uint8
        │  skeletonize + junction-guided refinement
        ▼
Skeleton + Junction set skeleton.npy, junctions.json
        │  node + edge extraction
        ▼
Raw Road Graph          graph_raw.gpkg  (nodes, edges, disconnected)
        │  confidence-guided healing (candidate → score → union-find → MST)
        ▼
Healed Road Graph       graph_healed.gpkg (connected)
        │  attribute assignment
        ▼
Weighted Attributed     graph.gpkg + graph.json
  Graph                 (length, dir, curvature, confidence, travel_cost)
        │  network intelligence
        ▼
Criticality Layers      centrality.json, bridges.json, articulation.json,
                        communities.json, criticality_score.json
        │  simulation (per scenario)
        ▼
Simulation Result       sim_{id}.json (resilience index, Δpath length,
                        efficiency, components, travel delay, reroutes)
        │  REST
        ▼
Dashboard               overlays + heatmaps + charts + interactions
```

Each artifact is content-addressed by `(scene_id, stage, params_hash)` and stored in object storage; the **Artifact Registry** (a small table) maps these keys to URIs. See §6.

---

## 4. Module map per plane

### 4.1 ML Inference Plane (`ml/`)
```
Input: raster URI + tiling config
  ├── data/tiler.py            windowed tiling, overlap, padding
  ├── data/transforms.py       normalize, augment (train), to-tensor
  ├── models/toposam_rr.py     encoder + decoder + heads  (see model.md)
  ├── losses/                  dice, lovasz, boundary, junction
  ├── engine/trainer.py        train loop, AMP, schedulers
  ├── inference/predictor.py   tiled inference + seam blending
  └── inference/stitch.py      reassemble tiles → full-scene prob maps
Output: road_prob.tif, junction.tif
```

### 4.2 Graph / Topology Plane (`graph/`)
```
Input: road_prob.tif, junction.tif
  ├── extraction/binarize.py        threshold, hysteresis, morphology
  ├── extraction/skeletonize.py     skeleton + junction-guided refinement
  ├── extraction/vectorize.py       nodes, edges, simplification
  ├── healing/candidates.py         endpoint pair generation (KD-tree)
  ├── healing/scoring.py            Connection Score engine
  ├── healing/optimizer.py          union-find + MST over candidates
  ├── attributes/assign.py          length, direction, curvature, cost
  └── analytics/                    centrality, bridges, articulation,
                                    communities, criticality_score
Output: graph.json/.gpkg + criticality layers
```

### 4.3 Simulation Plane (`simulation/`)
```
Input: attributed graph + scenario spec
  ├── scenarios/node_failure.py
  ├── scenarios/road_closure.py
  ├── scenarios/flooding.py         polygon → affected edges (shapely)
  ├── scenarios/congestion.py       cost multipliers
  ├── engine.py                     apply scenario → perturbed graph
  └── metrics/resilience.py         resilience index, efficiency, etc.
Output: sim_result.json
```

### 4.4 API Plane (`backend/`)
```
  ├── app/api/routes/               scenes, infer, graph, analytics, sim
  ├── app/services/                 orchestration over ml/graph/simulation
  ├── app/workers/tasks.py          Celery: infer_scene, build_graph, run_sim
  ├── app/core/                     config, logging, DI, artifact registry
  ├── app/db/                       registry + metadata (SQLite/Postgres)
  └── app/schemas/                  Pydantic request/response models
```

### 4.5 Presentation Plane (`frontend/`) — see [frontend.md](frontend.md).

---

## 5. Control flow: a request's life

```
User clicks "Analyze scene"  ──▶  POST /scenes/{id}/process
        │
        ▼
API creates Job, enqueues Celery: infer_scene → build_graph → analytics
        │  returns 202 + job_id
        ▼
Frontend polls GET /jobs/{job_id}  (or subscribes via SSE/WebSocket)
        │
        ▼  worker writes artifacts to MinIO, registers keys
Job = done  ──▶  Frontend GET /graph/{id}, GET /analytics/{id}
        │
        ▼
User drags a "flood polygon"  ──▶  POST /sim  {scenario, params}
        │  (fast, runs on cached graph, often < 1s, no GPU)
        ▼
Sim result returned synchronously  ──▶  dashboard updates overlays + charts
```

**Key latency design:** the *expensive* path (inference + graph build) runs once, async, cached. The *interactive* path (simulation, rerouting) operates on the cached in-memory graph and returns synchronously. This is what makes the dashboard feel real-time.

---

## 6. Artifact registry & caching

| Concept | Implementation |
|---------|----------------|
| Artifact key | `sha1(scene_id : stage : params_hash)` |
| Storage | MinIO/S3 bucket `route-resilience/{scene}/{stage}/...` |
| Registry | DB table `artifacts(key, scene_id, stage, uri, created_at, meta)` |
| Hot cache | Redis: serialized graph + centrality for active scenes (TTL) |
| Invalidation | Changing a stage's params → new `params_hash` → new key; old artifact retained (reproducibility) |

This registry is the backbone of both **scalability** (workers are stateless, read/write artifacts) and **reproducibility** (every result is traceable to inputs + params).

---

## 7. Scalability strategy (a headline feature)

| Concern | Mechanism | Why it scales |
|---------|-----------|---------------|
| Large rasters | Windowed tiling + per-tile inference | Memory bounded; tiles are embarrassingly parallel |
| GPU throughput | Celery worker pool with GPU concurrency limit | Add GPU workers → linear inference throughput |
| Heavy graph math | Cache centrality; igraph for hot path; per-component parallelism | Compute once, serve many; betweenness parallelizes over sources |
| Interactive sims | Run on cached in-memory graph, no recompute of upstream | O(query) not O(pipeline) |
| Many users | Stateless API replicas behind NGINX; shared Redis + object store | Horizontal scale of API tier |
| Bursty jobs | Redis-backed queue decouples submit from compute | Smooths spikes; backpressure visible |
| Storage growth | Object storage (S3/MinIO), not local disk | Effectively unbounded, cheap, versioned |
| Future cities | Scene-scoped keys; nothing global mutable | Add a city = add a key namespace |

**K8s readiness:** API `Deployment` (HPA on CPU/RPS), Worker `Deployment` (HPA on queue depth), Redis + MinIO as `StatefulSet`s, all configured via `ConfigMap`/`Secret`. The seams already exist because planes communicate via artifacts + jobs. See [deployment.md](deployment.md).

---

## 8. Configuration, logging, errors (cross-cutting)

- **Config:** layered YAML in `configs/` (`model.yaml`, `pipeline.yaml`, `app.yaml`) → loaded into Pydantic `Settings`, overridable by env vars. One source of truth, no magic constants in code.
- **Logging:** `structlog` JSON logs with a per-request/per-job `correlation_id` that threads API → worker → artifact. Makes the demo debuggable live.
- **Errors:** typed exceptions per plane (`InferenceError`, `GraphBuildError`, `SimError`) mapped to HTTP problem-details responses; jobs capture failures into the registry so the UI can show *why* a stage failed.
- **DI:** FastAPI dependency injection provides `Settings`, storage client, registry, and graph cache to routes/services — no global singletons, trivially testable.

---

## 9. Why this architecture for a 30-hour hackathon

1. **Parallelizable team work** — 5 planes, clean artifact contracts → 5 people build at once.
2. **De-risked demo** — cached artifacts mean the dashboard works even if the GPU dies or training stalls.
3. **Looks production-grade** — registry, jobs, caching, K8s manifests, structured logs — the *story* of scalability is real, not theatre.
4. **Incrementally truthful** — every stage degrades gracefully (e.g., healing falls back to plain MST; analytics falls back to degree centrality) so there's always *something* to show.

See [pipeline.md](pipeline.md) to descend into Stage 1–2, then [graph_engine.md](graph_engine.md) for Stage 3–5.
