# Backend Architecture

The backend is a **modular monolith** built on **FastAPI**, with a **Celery** worker for heavy/async work and **Redis** for caching + the job broker + live updates. It orchestrates the ML, graph, and simulation planes through a clean artifact contract.

---

## 1. Why this shape

| Decision | Choice | Why (for a 30h build) |
|----------|--------|------------------------|
| API style | **REST (FastAPI)** | Async, auto OpenAPI/Swagger (free demo UI), Pydantic validation, fastest to build |
| Topology | **Modular monolith**, not microservices | One process to run/debug; planes are *packages* with clean seams → split later without rewrite |
| Heavy work | **Celery worker + Redis broker** | Inference & graph build run off the request thread; API stays responsive |
| Cache / live | **Redis** | One dependency = cache + broker + pub/sub for SSE updates |
| Persistence | **SQLite → PostgreSQL/PostGIS** | SQLite for demo speed; Postgres+PostGIS path for real geo queries |
| Validation | **Pydantic v2** | Messy geo payloads validated at the boundary |

**REST vs GraphQL vs gRPC:** GraphQL's schema/resolver overhead buys nothing for this UI; gRPC complicates a browser client. REST+JSON with auto-docs is the pragmatic win.

**Microservices vs monolith:** the *logical* services are the five planes, but we deploy **two processes** (API + worker). The artifact-registry seam means any plane can later become its own service with no code rewrite — we get the scalability story without the distributed-systems tax during the hackathon. (See [architecture.md §2](architecture.md).)

---

## 2. Folder structure

```
backend/
├── app/
│   ├── main.py                 # FastAPI app factory, router mount, middleware
│   ├── api/
│   │   ├── deps.py             # DI providers (settings, storage, registry, graph cache)
│   │   └── routes/
│   │       ├── scenes.py       # upload/list/process scenes
│   │       ├── infer.py        # trigger inference, fetch prob maps
│   │       ├── graph.py        # fetch graph (GeoJSON/JSON)
│   │       ├── analytics.py    # criticality layers
│   │       ├── simulation.py   # run scenarios
│   │       └── jobs.py         # job status + SSE stream
│   ├── services/               # orchestration over ml/graph/simulation packages
│   │   ├── inference_service.py
│   │   ├── graph_service.py
│   │   ├── analytics_service.py
│   │   └── simulation_service.py
│   ├── workers/
│   │   ├── celery_app.py
│   │   └── tasks.py            # infer_scene, build_graph, run_analytics
│   ├── core/
│   │   ├── config.py           # Pydantic Settings (loads configs/*.yaml + env)
│   │   ├── logging.py          # structlog setup, correlation ids
│   │   ├── storage.py          # MinIO/S3 client (artifact get/put)
│   │   ├── registry.py         # artifact registry (key → uri)
│   │   ├── cache.py            # Redis graph cache (get/set serialized graph)
│   │   └── errors.py           # typed exceptions → HTTP problem details
│   ├── db/
│   │   ├── models.py           # Scene, Job, Artifact tables
│   │   └── session.py
│   └── schemas/                # Pydantic request/response models
└── tests/
```

## 3. Module communication & data flow

```
Route (thin)  ──▶  Service (orchestration)  ──▶  plane package (ml/graph/simulation)
     │                    │                              │
     │                    └── reads/writes Artifacts via registry + storage
     └── for heavy ops: enqueue Celery task, return 202 + job_id

Worker task ── runs plane code ── writes artifacts ── updates Job status ── publishes to Redis
```

- **Routes are thin** — parse/validate, call a service, shape the response. No business logic.
- **Services orchestrate** — decide cache hit vs compute, call plane packages, register artifacts.
- **Plane packages** (`ml/`, `graph/`, `simulation/`) are pure libraries — no web framework imports → unit-testable and reusable from CLI scripts.

## 4. Job lifecycle (async heavy work)

```
POST /scenes/{id}/process
  → service creates Job(status=queued), enqueues chain:
        infer_scene → build_graph → run_analytics
  → 202 {job_id}

worker executes chain, each step:
  - checks artifact cache (skip if hit)
  - computes, writes artifact, registers key
  - updates Job(status, progress, current_stage), publishes to Redis channel

GET /jobs/{job_id}            → current status (poll)
GET /jobs/{job_id}/stream     → SSE: live progress events
```

Idempotent by `params_hash`: re-processing an unchanged scene short-circuits on cache hits.

## 5. Caching strategy (three tiers)

| Tier | Store | Holds | TTL |
|------|-------|-------|-----|
| L1 in-process LRU | Python dict | Active scene's NetworkX graph object | process life |
| L2 Redis | Redis | Serialized graph + criticality JSON + sim results | hours |
| L3 object storage | MinIO/S3 | All artifacts (`road_prob.tif`, `graph.gpkg`, …) | durable |

Simulation reads the graph from L1/L2 (no disk), which is why scenarios are sub-second. Inference/graph artifacts live in L3, registered for reproducibility.

## 6. Configuration management

- `core/config.py` → Pydantic `Settings` loads `configs/app.yaml`, `model.yaml`, `pipeline.yaml`, overridable by env (`RR_` prefix) and `.env`.
- One typed `Settings` injected via DI — **no magic constants in code**, and every pipeline parameter feeds the artifact `params_hash`.

## 7. Dependency injection
FastAPI `Depends` provides `Settings`, `StorageClient`, `ArtifactRegistry`, `GraphCache`, DB session to routes/services. Swappable in tests (e.g. in-memory storage, fake registry) → fast, isolated unit tests. No global singletons.

## 8. Logging & error handling
- **structlog** JSON logs; a `correlation_id` is generated per request and propagated into Celery tasks → one id threads API → worker → artifact, making the live demo debuggable.
- **Typed exceptions** (`InferenceError`, `GraphBuildError`, `SimError`, `NotFound`) caught by an exception handler → RFC-7807 problem-details JSON with the correlation id. Failed jobs store the error so the UI shows *why* a stage failed, not a blank spinner.

## 9. GPU inference pipeline (backend view)
- The Celery worker holds the model in memory (lazy-loaded singleton) so weights load once, not per request.
- GPU concurrency is bounded (`worker_concurrency` + a GPU semaphore) to avoid OOM; extra jobs queue in Redis.
- Inference writes `road_prob.tif`/`junction.tif` to object storage and registers them → downstream graph build reads from storage, decoupled from the GPU.
- CPU-only deployments swap to the ONNX/U-Net path transparently (same task interface).

## 10. Graph computation pipeline (backend view)
- `graph_service` loads cached graph if present (L1/L2); else runs `graph/` package on the inference artifacts and caches the result.
- Analytics (centrality etc.) computed in the worker (can be slow on big graphs), cached as JSON; the API serves them instantly thereafter.
- Simulation runs **in the API process** (fast, on cached in-memory graph) for low latency, or in the worker for very large graphs.

## 11. Security & limits (lightweight, demo-appropriate)
- Request size limits on raster upload; presigned object-storage URLs for large files.
- CORS locked to the frontend origin; optional API key for write endpoints.
- Rate limiting on `/sim` (cheap but spammable) via Redis token bucket.

## 12. Testing
- Plane packages: pure unit tests (no server).
- Services: tested with fake storage/registry via DI.
- API: `pytest` + `httpx` against the app with a fake worker (synchronous task mode).
- See [api.md](api.md) for the endpoint contracts these tests assert.
