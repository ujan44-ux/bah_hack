# Route Resilience — Hackathon Implementation Bible (v0.1)

> **Production-first architecture, hackathon-first implementation.**
> The MVP implements *fewer modules of the same architecture* — not a throwaway prototype. Every line written in the 30 hours survives into the Production SaaS (v1.0). The only thing that changes later is *which implementation sits behind an interface*, never the interface itself.

This file is the team's single source of truth. Read [docs/architecture.md](docs/architecture.md) once for the full design; live in **this** file during the build.

---

## 0. Project Goal

Turn a raw satellite raster of a city into a **connected, weighted, stress-testable road graph** with a **criticality heatmap** and **live failure simulation**, presented in an interactive planner dashboard — robust to roads hidden under canopy, shadow, and clouds.

**One sentence:** *We don't just segment roads; we rebuild the city's road network as a graph and tell planners which roads will break it if they fail.*

---

## 1. Core USP (the things that MUST be real)

These five are the project. If anything else slips, these do not:

1. **Occlusion-robust road + junction extraction** (TopoSAM-RR: frozen SAM + decoder + dual heads).
2. **Confidence-guided graph healing** (Connection Score → MST) — the signature algorithm that reconnects broken roads using *model belief*, not blind geometry.
3. **Criticality analysis** (betweenness + bridges + articulation + communities → Urban Criticality Score).
4. **Failure simulation + resilience index** (node/road/flood/congestion → efficiency drop, reroute, delay).
5. **Interactive dashboard** that makes all of the above legible to a non-technical planner.

Everything else exists to support these five.

---

## TASK 1 — Module Classification (nothing disappears)

Legend: **Build** = real code for judging · **Mock** = realistic fake output behind the real interface · **Stub** = folder/interface only · **Prod** = documented, not built.

| Module | In Architecture | Hackathon action | Mock? | Future SaaS |
|--------|:---:|---|:---:|---|
| Raster reader / tiler (`ml/data`) | ✅ | **Build** | — | same, + windowed COG streaming |
| TopoSAM-RR model (`ml/models`) | ✅ | **Build** (frozen SAM + decoder + heads) | — | unfreeze / SAM2 / multimodal |
| Training engine (`ml/engine`) | ✅ | **Build (light)** — fine-tune script | — | full MLOps, sweeps, registry |
| Inference predictor + stitch (`ml/inference`) | ✅ | **Build** | — | batched cloud GPU workers |
| Binarize/skeleton/vectorize (`graph/extraction`) | ✅ | **Build** | — | same |
| Healing: candidates/scoring/MST (`graph/healing`) | ✅ | **Build** | — | learned GNN scorer (drop-in) |
| Attribute assignment (`graph/attributes`) | ✅ | **Build** | — | road-class head, lanes, direction |
| Analytics (`graph/analytics`) | ✅ | **Build** | — | demand-weighted, igraph accel |
| Simulation engine + metrics (`simulation`) | ✅ | **Build** | — | cascading/traffic-assignment |
| FastAPI app + routes (`backend/app/api`) | ✅ | **Build** | — | same, + auth/gateway |
| Services layer (`backend/app/services`) | ✅ | **Build** | — | same interfaces, scaled impls |
| Artifact registry (`core/registry.py`) | ✅ | **Build (SQLite/JSON impl)** | — | Postgres rows, unchanged interface |
| Storage client (`core/storage.py`) | ✅ | **Build (local FS impl of S3 interface)** | — | swap to MinIO/S3 — same interface |
| Cache (`core/cache.py`) | ✅ | **Build (in-process LRU + dict)** | — | swap to Redis — same interface |
| Job runner (`workers/`) | ✅ | **Build (in-process BackgroundTasks behind a JobRunner interface)** | — | swap to Celery — same interface |
| Config / logging / errors (`core/`) | ✅ | **Build** | — | same |
| Frontend dashboard (`frontend/`) | ✅ | **Build** | — | multi-city/user, saved sims |
| SSE job progress | ✅ | **Build (simple)** | — | same |
| Auth / multi-user | ✅ | **Stub** (single demo user, interface present) | — | OAuth/JWT impl |
| Object storage (MinIO/S3) | ✅ | **Stub** (local FS adapter implements it) | — | real bucket |
| Redis | ✅ | **Stub** (cache interface, dict impl) | — | real Redis |
| Celery / queue | ✅ | **Stub** (JobRunner interface, sync impl) | — | real broker |
| Kubernetes manifests | ✅ | **Prod** (write manifests, don't run) | — | deploy |
| Postgres/PostGIS | ✅ | **Stub** (SQLAlchemy models, SQLite URL) | — | flip the DB URL |
| Monitoring (Prometheus/Grafana) | ✅ | **Prod** (documented) | — | wire up |
| Model registry/versioning | ✅ | **Build (light)** — checkpoint tag + meta JSON | — | full registry + canary |
| Population / demand layers | ✅ | **Mock** (synthetic O-D or uniform) | ✅ | real census/POI data |
| Historical traffic / weather APIs | ✅ | **Mock** (static JSON) | ✅ | live API integrations |
| ISRO/large datasets | ✅ | **Mock** (one curated demo scene) | ✅ | full ingestion |

**Principle:** every "future" item is reachable by swapping an *implementation behind an interface we build now*. No row says "rewrite."

---

## TASK 2 — MVP Implementation Architecture (with extension points)

What actually runs in the 30h demo — a **single FastAPI process** + worker-as-background-task + Next.js. Dashed boxes are *interfaces present, simple implementation now, scaled implementation later*.

```
┌────────────────────────────────────────────────────────────────────────┐
│                         NEXT.JS DASHBOARD                               │
│   MapLibre + deck.gl · scenario builder · metrics · resilience curve   │
└───────────────────────────────┬────────────────────────────────────────┘
                                │ REST + SSE
┌───────────────────────────────▼────────────────────────────────────────┐
│                       FASTAPI APP (single process)                      │
│                                                                         │
│   api/routes ── services ──┬── InferenceService ──► ml/  (TopoSAM-RR)   │
│        │                   ├── GraphService     ──► graph/ (heal/attrs) │
│        │                   ├── AnalyticsService ──► graph/analytics     │
│        │                   └── SimulationService──► simulation/         │
│        │                                                                │
│   ┌────┴───────────── CORE INTERFACES (swap point) ──────────────────┐  │
│   │ JobRunner  : BackgroundTasks  ┄┄┄►  [Celery]                     │  │
│   │ Storage    : LocalFS adapter  ┄┄┄►  [MinIO/S3]                   │  │
│   │ Cache      : in-proc LRU/dict ┄┄┄►  [Redis]                      │  │
│   │ Registry   : SQLite/JSON      ┄┄┄►  [Postgres/PostGIS]           │  │
│   │ Auth       : single demo user ┄┄┄►  [OAuth/JWT]   (stub)         │  │
│   └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                 ┌──────────────▼───────────────┐
                 │  Local artifact dir (./data)  │  ┄┄┄►  [Object storage]
                 │  road_prob.tif · graph.json   │
                 │  criticality.json · sim/*.json│
                 └───────────────────────────────┘

Future extension points (interfaces exist, not wired in v0.1):
  · API Gateway / Auth  in front of FastAPI
  · Celery workers      behind JobRunner
  · Redis               behind Cache
  · S3/MinIO            behind Storage
  · Monitoring          around the app (structlog already emits JSON)
  · K8s                 wraps the same containers
```

**The whole trick:** the *services* and *core interfaces* are identical to v1.0. We ship the simplest implementation of each interface (local FS, dict cache, sync jobs) and the production version is a constructor swap chosen by config.

---

## TASK 3 — Preserve Scalability by Design (swap table)

| Concern | Hackathon impl | Future impl | How it swaps without architecture change |
|---------|----------------|-------------|------------------------------------------|
| Compute orchestration | `JobRunner` = FastAPI `BackgroundTasks` (sync chain) | Celery + Redis broker | Same `JobRunner.enqueue(task, args)` interface; DI picks impl from `app.yaml` |
| Caching | `Cache` = in-process LRU + dict | Redis | Same `Cache.get/set`; flip `cache.backend: redis` |
| Storage | `Storage` = local FS adapter (`./data/...`) implementing S3-style `put/get/url` | MinIO/S3 | Same `Storage` interface; flip `storage.backend: minio` |
| Registry/metadata | SQLite via SQLAlchemy | Postgres/PostGIS | Change DB URL only; models unchanged |
| Multi-user | single demo user, `current_user` dependency returns a fixed user | OAuth/JWT provider | Replace the `get_current_user` dependency impl |
| Deployment | `uvicorn` + `next dev` (or compose) | K8s (HPA on queue depth) | Containers identical; manifests already written |
| GPU scaling | one local GPU, in-proc inference | GPU worker pool | Inference already behind `InferenceService` + `JobRunner` |
| Monitoring | structlog JSON to stdout | Loki/Prometheus/Grafana | Logs already structured + correlation ids |

**Rule of thumb for the team:** *never call Redis/S3/Celery directly.* Always go through `Cache`, `Storage`, `JobRunner`. That single discipline is what makes v0.1 → v1.0 a config change.

---

## TASK 4 — Upgrade Path (the architecture grows, never restarts)

```
v0.1  HACKATHON MVP (30h)
  · Single FastAPI process + BackgroundTasks
  · Local FS storage, dict cache, SQLite registry
  · TopoSAM-RR (frozen SAM) on one GPU, one demo city
  · Full graph → heal → analytics → simulation → dashboard
        │
        ▼
v0.5  HARDENED PROTOTYPE (post-hackathon, ~weeks)
  · Swap JobRunner→Celery, Cache→Redis, Storage→MinIO  (config flips)
  · docker-compose all services; SSE → robust job tracking
  · Add auth (single org), persist multiple scenes
  · Eval harness + model registry (checkpoint tags)
        │
        ▼
v1.0  PRODUCTION SaaS (multi-tenant)
  · Postgres+PostGIS; multi-user/multi-city; OAuth
  · K8s: API replicas + GPU worker autoscaling (HPA/KEDA)
  · Object storage + CDN tiles; monitoring (Prometheus/Grafana)
  · Saved simulations, historical comparisons, report export
        │
        ▼
v2.0  GEOSPATIAL PLATFORM
  · SAM2/multimodal (SAR+DEM), temporal fusion, learned GNN healing
  · Demand-weighted criticality, cascading-failure simulation
  · Optimization (which roads to reinforce), active-learning loop
  · QGIS plugin / OSM export, disaster-response real-time mode
```

Each arrow **adds or swaps implementations behind existing interfaces** — no plane is rebuilt.

---

## TASK 5 — Per-module classification (Build Now / Stub / Mock / Prod-only)

**BUILD NOW (must exist for judging):**
`ml/data` · `ml/models/toposam_rr` · `ml/inference` · `graph/extraction` · `graph/healing` · `graph/attributes` · `graph/analytics` · `simulation` · `backend/app/api` + `services` + `core` (config/logging/errors + LocalFS Storage + dict Cache + sync JobRunner + SQLite Registry) · `frontend` dashboard.

**STUB INTERFACE (folder/interface only, no impl):**
`core/auth.py` (returns demo user) · `workers/celery_app.py` (interface present, sync impl active) · Redis adapter class (unused) · MinIO adapter class (unused) · Postgres path (models exist, SQLite URL active).

**MOCK (realistic fake outputs behind real interface):**
Population/demand layer → synthetic uniform or hand-placed O-D pairs · historical traffic/weather → static JSON · "large dataset ingestion" → one curated demo scene + pre-baked inference cache (Tier-0).

**PRODUCTION ONLY (documented, not built):**
Kubernetes apply · Prometheus/Grafana wiring · API gateway · multi-tenant billing · CDN · canary model rollout.

---

## TASK 6 / Features summary

### Features to BUILD (real)
Road extraction · junction extraction · skeleton→graph · **confidence-guided healing** · attributes/weights · betweenness/bridges/articulation/communities · **criticality score** · node failure · road closure · **flood-region simulation** · reroute + resilience index · interactive map dashboard with heatmap + scenario controls.

### Features to SKIP (architecture-only this round)
K8s deploy · Prometheus · multi-tenant auth · billing · CDN · model canary · distributed training.

### Features to MOCK
Population/demand weighting · historical traffic · weather/flood depth data (user draws polygon instead) · cloud storage (local FS) · large multi-city ingestion.

### Features to KEEP REAL (the USP — non-negotiable)
The five Core USP items in §1.

---

## TASK 6 — Build Order (dependency-aware)

> **Golden path:** build the **Tier-0 cached demo** first so the dashboard works end-to-end on a pre-baked `road_prob.tif` *before* the model is trained. Everything downstream then progresses in parallel.

```
0. Repo + core interfaces (Storage/Cache/JobRunner/Registry) + config   [hour 0–2]
1. TIER-0: drop one pre-baked road_prob.tif into ./data, wire artifact registry
2. graph/extraction (binarize→skeleton→vectorize)   ──► raw graph
3. graph/healing (candidates→scoring→MST)            ──► connected graph   ◄ USP
4. graph/attributes + analytics (criticality)        ──► criticality layers ◄ USP
5. simulation engine + metrics                       ──► resilience          ◄ USP
6. FastAPI routes/services over 2–5                   ──► API
7. frontend: map + graph overlay → criticality → scenario builder → charts
8. ml: train TopoSAM-RR in parallel; swap real inference for Tier-0 cache when ready ◄ USP
9. polish: demo scene, healed-edge styling, reroute animation, story slide
```

Stages 2–7 do **not** wait for stage 8. The artifact contract (`road_prob.tif`) decouples them.

---

## TASK 6 — Team Allocation (4 members, maximize parallelism)

| Member | Owns | Parallel from hour |
|--------|------|--------------------|
| **A — ML / Perception** | `ml/` TopoSAM-RR, training, inference, stitch; produces real `road_prob.tif`/`junction.tif`. Provides **Tier-0 cached artifact on hour 1** so others aren't blocked. | 0 |
| **B — Graph Engine** | `graph/` extraction → healing → attributes → analytics. The USP algorithms. Works off Tier-0 cache immediately. | 1 (after Tier-0) |
| **C — Backend / Integration** | `backend/` core interfaces (Storage/Cache/JobRunner/Registry), services, routes, SSE, config, the artifact contract. Unblocks A↔B↔D. | 0 |
| **D — Frontend / Demo** | `frontend/` dashboard, map layers, scenario builder, charts, demo polish, slides. Builds against mocked API first, then real. | 0 (with API mocks) |

**Critical handshakes:** C defines the JSON schemas (graph.json, criticality.json, sim_result.json) in **hour 1** so B and D code against a frozen contract. A delivers Tier-0 cache in **hour 1** so B starts immediately. Simulation (§USP) is co-owned B+C.

---

## TASK 6 — 30-Hour Timeline

| Window | Milestone | Owners |
|--------|-----------|--------|
| **0–2h** | Repo cloned, deps installed, core interfaces + config + JSON schemas frozen. Tier-0 `road_prob.tif` in `./data`. "Hello graph" renders on map. | All / A,C |
| **2–6h** | `graph/extraction` → raw graph from cache. `graph/healing` MVP (distance+angle scoring) → connected graph. API `GET /graph` live. Map shows roads. | B,C,D |
| **6–10h** | Full Connection Score (+seg/junction conf). Attributes + betweenness/bridges/articulation. `GET /analytics`. Criticality heatmap on map. | B,C,D |
| **10–14h** | Simulation engine: node/road/flood + resilience metrics. `POST /simulate`. Scenario builder + flood-polygon draw in UI. | B,C,D |
| **14–18h** | Reroute engine + before/after animation. Resilience curve (targeted vs random). Metric cards + charts. **End-to-end demo on cached data works.** | C,D,B |
| **18–22h** | A's trained TopoSAM-RR swapped in for real inference; validate on held-out demo scene. Healed edges styled distinctly. SSE job progress. | A,C |
| **22–26h** | Polish: tooltips, layer toggles, second demo scene, edge cases, error states. Eval report (`scripts/evaluate.py`). | All |
| **26–28h** | **Freeze code.** Rehearse demo twice. Record a backup screen-capture of the full flow. | All |
| **28–30h** | Slides, elevator pitch, judge-Q prep, deploy compose on demo machine, final dry run. | All |

**Hard rule:** code freeze at hour 26. Hours 26–30 are demo insurance, not features.

---

## TASK 6 — Mock Strategy (exactly what's faked and how)

| Thing | Why mock | Mock implementation |
|-------|----------|---------------------|
| Large ISRO/national datasets | Too big, irrelevant to judging the algorithm | One curated demo city tile + a held-out scene |
| Pre-trained model availability | De-risk GPU/training | **Tier-0**: pre-baked `road_prob.tif` in `./data`, registered as an artifact — dashboard runs on it |
| Population / travel demand | No clean source in 30h | Synthetic: uniform O-D, or a few hand-placed key trips (hospital↔zones) |
| Historical traffic | No feed | Static JSON congestion multipliers per edge (optional layer) |
| Weather / flood depth | No API | **User draws the flood polygon** on the map — more interactive *and* avoids the API |
| Cloud storage | No infra | `Storage` interface with LocalFS adapter (`./data`) |
| Auth / multi-user | Not judged | `get_current_user` returns a fixed demo user |
| Celery/Redis | Not judged at MVP scale | `JobRunner`/`Cache` interfaces, sync + dict impls |

Every mock sits **behind the real interface**, so replacing it later is a swap, not a rewrite.

---

## TASK 6 — Real Components (the USP — fully implemented)

Already enumerated in §1. Restated as the build contract: **road extraction, graph generation, graph healing, criticality analysis, failure simulation, interactive visualization are REAL.** If forced to cut, cut polish/second-scene/SSE — never these.

---

## TASK 6 — Demo Script (minute-by-minute, ~6 min)

| Time | Action | Narration |
|------|--------|-----------|
| 0:00 | Open dashboard, select demo city | "A satellite view of [city]. Roads here vanish under canopy and shadow." |
| 0:30 | Toggle raw segmentation vs healed graph | "Naive segmentation gives a *fragmented* mask — useless for routing. Watch what we do." |
| 1:00 | Show healed graph; highlight **healed edges** in a distinct color | "Our Connection Score reconnects breaks using the model's *own confidence* in the gap — not blind nearest-neighbor. These dashed segments are AI-healed." |
| 2:00 | Toggle **criticality heatmap** | "Now the network analysis: a handful of arterial roads carry most trips. These red ones are bridges and articulation points — single points of failure." |
| 3:00 | **Draw a flood polygon** over a river district → run | "Suppose the river floods this district…" |
| 3:30 | Metrics update: resilience index ↓, components ↑, affected roads glow | "Network efficiency drops 38%, the city splits into 3 disconnected pieces." |
| 4:00 | Pick **ambulance O-D** → before/after route animates | "An ambulance route that was 8 minutes is now 14 — and these neighborhoods are unreachable." |
| 4:45 | Run **resilience curve**: targeted vs random | "When we remove roads *by our criticality score*, the network collapses far faster than random — proof the score finds the roads that actually matter." |
| 5:30 | Close on architecture/scale slide | "Same architecture scales to any city on cloud GPUs — this is v0.1 of a planning platform." |

---

## TASK 6 — Judging Story (the narrative arc)

Problem (fragmented masks are unusable) → Insight (treat the **graph** as the product) → Method (occlusion-robust extraction + **confidence-guided healing**) → Payoff (criticality + **failure simulation** answers the planner's real question) → Proof (resilience curve self-validates) → Scale (production-first architecture). One coherent story, each USP a beat in it.

---

## TASK 6 — Deployment Strategy (simplest thing that looks professional)

- **Demo machine:** `docker compose up` brings api + frontend (+ optional minio/redis if time). Fallback: `uvicorn` + `next dev` directly.
- **Looks professional because:** auto Swagger at `/docs`, structured JSON logs with correlation ids, `/health` + `/version` endpoints, K8s manifests committed (shown, not run).
- **Pre-seed** the demo scene + cached artifacts via `scripts/demo_seed.py` so cold-start is instant.
- **Backup:** a recorded screen-capture of the full demo flow (hour-26 deliverable).

---

## TASK 6 — Risk Register

| Risk | Likelihood | Impact | Mitigation | Fallback |
|------|:---:|:---:|-----------|----------|
| Training doesn't converge in time | Med | High | Tier-0 cache from hour 1; checkpoint by APLS-lite | Demo on cached `road_prob.tif`; mention model is "fine-tuning live" |
| GPU unavailable at demo | Low | High | ONNX/CPU path; cached inference | Tier-0 cache, zero GPU needed |
| Healing produces garbage links | Med | High | Score thresholds + `fallback: mst`; flag healed edges | Plain Euclidean MST — still connected |
| Graph too big → slow centrality | Med | Med | Sampled-k betweenness; cache results | Pre-compute analytics offline, serve JSON |
| Frontend perf with many edges | Low | Med | deck.gl GPU rendering; simplify polylines | Cap rendered edges by criticality |
| Integration drift (schema mismatch) | Med | High | Freeze JSON schemas hour 1; zod/pydantic validation | C owns contract, single source |
| Time overrun | High | Med | Code freeze hour 26; cut polish first | Recorded demo |
| Live demo network/crash | Low | High | Run fully local; pre-seeded | Play recorded capture |

---

## TASK 6 — Expected Judge Questions + Suggested Answers

**Q: How is this different from existing road segmentation (DeepGlobe/SpaceNet models)?**
A: Those stop at a pixel mask. We treat segmentation as stage 1 of 7 and make the *routable graph* the product — including confidence-guided healing of occluded gaps and graph-theoretic criticality. We optimize APLS (topology), not just IoU.

**Q: Why freeze the SAM encoder — isn't fine-tuning better?**
A: For a 30h budget, frozen SAM gives occlusion-robust features trained on 1B masks for free; we train only ~5–10M decoder/head params and converge in hours on one GPU. The architecture supports unfreezing/SAM2 later with no interface change. We also keep a U-Net fallback.

**Q: How do you know the healed roads are real, not hallucinated?**
A: Three safeguards — (1) the Connection Score requires *segmentation evidence in the gap*, so we only connect where the model faintly sees road; (2) every healed edge is flagged and rendered distinctly — we never pass inferred links off as observed; (3) we report healed-edge precision against ground truth.

**Q: Does the criticality score actually identify the right roads?**
A: Yes, and we prove it live: the targeted-vs-random resilience curve. Removing roads by our score collapses network efficiency far faster than random removal — the definition of having found the critical ones.

**Q: Is this just a hackathon prototype?**
A: No — it's v0.1 of a production architecture. Every external dependency (storage, cache, jobs, DB, auth) is behind an interface; going to cloud SaaS is a config swap, not a rewrite. The K8s manifests are already written.

**Q: How does it scale to a whole country?**
A: Tiles are embarrassingly parallel → add GPU workers (HPA on queue depth). Analytics are cached per scene. Scenes are key-namespaced, nothing global mutable. See deployment.md.

**Q: What about clouds/weather/SAR?**
A: v2.0 roadmap — multimodal (SAR penetrates cloud) and temporal fusion plug into the same encoder interface.

---

## TASK 7 — Repository guidance (keep it production-shaped)

| Folder | Contains | Status |
|--------|----------|--------|
| `ml/` | **Real code** — model, train, inference | Build |
| `graph/` | **Real code** — the USP algorithms | Build |
| `simulation/` | **Real code** | Build |
| `backend/app/services`, `api`, `core` | **Real code** + the swappable interfaces | Build |
| `backend/app/core/{storage,cache,jobrunner,registry}.py` | Real interface + simple impl now; prod impl later | Build (interface), Stub (prod adapters) |
| `backend/app/core/auth.py`, `workers/celery_app.py` | **Placeholder interfaces** | Stub |
| `frontend/` | **Real code** — demo-winning UI, scalable component tree | Build |
| `configs/` | **Config** — model/pipeline/app YAML (driving the impl swaps) | Build |
| `deployment/docker`, `compose` | **Deployment** — used in demo | Build |
| `deployment/k8s` | **Deployment** — committed, shown, not run | Prod-only |
| `docs/` | **Documentation** — the full design suite | Done |
| `scripts/` | demo_seed, infer_scene, build_dataset, evaluate | Build (light) |
| `data/`, `artifacts/` | **Demo assets** — pre-baked scene + cached artifacts | Seed |
| `tests/` | unit (plane packages), integration (API), a couple e2e | Build (light) |

Do **not** flatten into one `app.py`. The folder structure *is* the scalability story; keep it.

---

## TASK 8 — Frontend review (demo-optimized, future-proof)

- **Demo-optimized:** MapLibre + deck.gl (GPU, smooth with 100k edges), scenario builder, reroute animation, resilience curve — the visual punch that wins.
- **Future-proof without redesign** because:
  - **Scene-scoped state** (`sceneId` in the store) → multi-city is "fetch a different scene," not a rewrite.
  - **TanStack Query** server-state layer → cloud inference / async jobs already modeled as queries+mutations.
  - **Zustand store keyed by scene/scenario** → saved simulations = persist the scenario spec; historical comparison = two scene ids side-by-side (the `<Compare>` split already exists conceptually).
  - **Typed API layer (zod)** → multi-user/auth adds a token header in one place.
- v0.1 ships single-city/single-user, but the component tree (TopBar→SceneSelector, MapView→layers, panels) needs **no structural change** for the v1.0 features.

---

## TASK 9 — Backend review (modular monolith, future services)

Single FastAPI process, but organized as the future services:
```
backend/app/
  api/routes/{scenes,infer,graph,analytics,simulation,jobs}.py   # thin
  services/{inference,graph,analytics,simulation}_service.py     # orchestration
  core/{config,logging,errors,storage,cache,jobrunner,registry,auth}.py  # interfaces
  workers/{celery_app,tasks}.py                                  # JobRunner impls
  db/{models,session}.py                                         # SQLite→Postgres
  schemas/                                                       # pydantic contracts
```
- **Clean interfaces** (`Storage`, `Cache`, `JobRunner`, `Registry`, `get_current_user`) injected via FastAPI `Depends` → swap impl per config, test with fakes.
- **Plane packages** (`ml/`, `graph/`, `simulation/`) import-only libraries, no web deps → reusable from CLI and future microservices.
- Each `services/*_service.py` is a future microservice boundary already.

---

## TASK 10 — Final Architecture Review (honest assessment)

**1. Does the MVP preserve the long-term architecture?**
Yes. The five planes, the service boundaries, and the four swap-interfaces (Storage/Cache/JobRunner/Registry) are identical to v1.0. The MVP implements the *simple* side of each interface.

**2. Which modules are intentionally left unimplemented?**
Auth/multi-tenancy, Celery/Redis (interfaces only), MinIO/S3 (LocalFS adapter stands in), Postgres/PostGIS (SQLite stands in), K8s/monitoring (committed/documented), demand/traffic/weather data (mocked).

**3. What hackathon code survives unchanged into the SaaS?**
All of `ml/`, `graph/`, `simulation/` (pure libraries); the `services/` orchestration; the API routes + pydantic schemas; the frontend component tree + state model; the config system. This is the majority of the code and *all* of the USP.

**4. What gets replaced later?**
Only the *implementations behind interfaces*: dict-cache→Redis, LocalFS→S3, sync-JobRunner→Celery, SQLite→Postgres, demo-user→OAuth. Each is a single class swap chosen by config.

**5. Does it avoid technical debt?**
Largely, *if* the team honors one discipline: **never call Redis/S3/Celery/DB directly — always through the interface.** The main residual debt risk is the in-process JobRunner hiding concurrency assumptions; we mitigate by keeping task functions pure and idempotent (keyed by `params_hash`) from day one.

**6. Could a startup continue from this repo?**
Yes. The repo already has plane separation, interface seams, config-driven impls, deployment manifests, and a documentation suite. v0.5 (compose + Redis/Celery/MinIO) is days of work, not a rewrite.

**Recommended refinements to lock in now (cheap, high-leverage):**
- Define the **four core interfaces as ABCs** in `core/` on hour 0, with the simple impls — this is the entire anti-debt strategy in one move.
- **Freeze the JSON schemas** (graph/criticality/sim) hour 1; validate with pydantic+zod both sides.
- Make every worker task **idempotent + keyed by `params_hash`** from the start so Celery is a literal drop-in.
- Keep healed edges **flagged everywhere** (data + UI) — protects credibility and is free.

---

## Testing Checklist
- [ ] `graph/extraction` produces a non-empty graph on the demo scene
- [ ] Healing reduces connected components toward 1 (assert before>after)
- [ ] Connection Score falls back to MST when scores degenerate
- [ ] Analytics: bridges/articulation found on a hand-checked toy graph
- [ ] Simulation: removing a known bridge raises components (unit test)
- [ ] Resilience index ∈ [0,1]; efficiency finite under disconnection
- [ ] API contract: graph/analytics/sim responses validate against schemas
- [ ] Frontend renders graph + heatmap; flood draw → metrics update
- [ ] End-to-end on Tier-0 cache with zero GPU
- [ ] Real inference swapped in matches cache-path contract

## Deployment Checklist
- [ ] `docker compose up` brings api + frontend (+ optional redis/minio)
- [ ] `scripts/demo_seed.py` loads demo scene + cached artifacts
- [ ] `/health`, `/ready`, `/version` green
- [ ] `/docs` Swagger loads
- [ ] Frontend points at API base URL via env
- [ ] Recorded backup demo capture saved locally

## Presentation Checklist
- [ ] Slides: problem → graph insight → healing → criticality → simulation → scale
- [ ] Elevator pitch memorized (below)
- [ ] Demo rehearsed twice; backup video ready
- [ ] Judge-Q answers reviewed by whole team
- [ ] One architecture slide showing v0.1→v1.0 swap story
- [ ] Eval report (APLS, healed-edge precision, resilience AUC) on a slide

## Elevator Pitch
> *"Satellite road maps fall apart under trees and shadows, and a broken map can't route an ambulance. Route Resilience rebuilds the city's roads as a connected graph — healing the gaps using the AI's own confidence — then tells planners exactly which roads, if they fail or flood, will cut the city in half. It's built production-first: this hackathon version is v0.1 of a planning platform that scales to any city on the cloud."*

## Stretch Goals (only if time remains)
- [ ] Second demo city to show generality
- [ ] SSE live pipeline progress bar
- [ ] Targeted-vs-random resilience curve animation
- [ ] GeoPackage export button (GIS interoperability)
- [ ] Congestion + flood *stacked* scenario
- [ ] Swap dict-cache → real Redis to flex the scale story live

---

### The one rule that makes all of this true
**Always go through the interface (`Storage`, `Cache`, `JobRunner`, `Registry`, `get_current_user`). Never call the backing service directly.** That single discipline is what turns a 30-hour MVP into the first commit of a production SaaS.
