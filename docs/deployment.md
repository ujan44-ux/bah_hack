# Deployment & Scalability

Two deployment targets: **(1) docker-compose** for the hackathon demo (one command, runs on a laptop), and **(2) Kubernetes manifests** that prove the production/scale story. The architecture's artifact-contract seams (see [architecture.md](architecture.md)) make the jump from one to the other a matter of *replicas*, not rewrites.

---

## 1. Demo deployment — docker-compose

```
deployment/compose/docker-compose.yml
services:
  api       FastAPI (uvicorn)         :8000   depends_on redis, minio
  worker    Celery (ML+graph)         GPU      depends_on redis, minio
  redis     cache + broker + pubsub   :6379
  minio     S3-compatible storage     :9000/:9001
  frontend  Next.js                   :3000
  nginx     reverse proxy / TLS       :80/:443
```

```bash
docker compose -f deployment/compose/docker-compose.yml up --build
python scripts/demo_seed.py --city demo_city   # loads pre-baked tiles + checkpoint
open http://localhost:3000
```

- **GPU**: worker uses `--gpus all` (NVIDIA Container Toolkit). If no GPU, set `RR_MODEL_DEVICE=cpu` → ONNX/U-Net fallback path.
- **Demo safety**: `demo_seed.py` pre-loads cached artifacts into MinIO so the dashboard works even if the GPU/model is unavailable. **This is the demo's insurance policy.**

## 2. Images — `deployment/docker/`

| Image | Base | Notes |
|-------|------|-------|
| `api.Dockerfile` | `python:3.11-slim` | FastAPI + plane packages; multi-stage, no torch CUDA needed |
| `worker.Dockerfile` | `pytorch/pytorch:cuda` | Model + graph code; heavy, GPU |
| `frontend.Dockerfile` | `node:20` → static export | multi-stage build → tiny runtime |

Multi-stage builds keep runtime images lean; deps pinned via `requirements.txt` / `package-lock.json` for reproducible builds.

## 3. Production deployment — Kubernetes (`deployment/k8s/`)

```
            ┌──────────── Ingress (NGINX) ────────────┐
            │                                          │
      Service: frontend                         Service: api  (HPA: CPU/RPS)
            │                                          │
            └───────── Deployment: api (N replicas, stateless) ──────┐
                                                                     │ enqueue
                                                              Service: redis (StatefulSet)
                                                                     │ dequeue
                              Deployment: worker (HPA: queue depth, GPU node pool)
                                                                     │
                                                          Service: minio (StatefulSet, PVC)
                                                                     │
                                              (prod) Postgres+PostGIS (StatefulSet, PVC)
```

| Component | K8s object | Scaling signal |
|-----------|-----------|----------------|
| api | Deployment + HPA | CPU / requests-per-second |
| worker | Deployment + HPA | **Redis queue depth** (KEDA scaler) |
| redis | StatefulSet | vertical; or managed Redis |
| minio | StatefulSet + PVC | or managed S3 |
| frontend | Deployment | static, trivially replicated |
| config | ConfigMap (`configs/*.yaml`) + Secret (keys) | — |

GPU workers run on a tainted GPU node pool; CPU workers (graph/sim only) can scale separately on cheap nodes.

## 4. Why this scales (the headline feature)

| Property | Mechanism | Result |
|----------|-----------|--------|
| **Stateless API** | no in-process session state; graph cache in Redis/object store | scale API horizontally behind the ingress |
| **Decoupled compute** | Redis queue between submit and worker | bursts absorbed; submit never blocks on GPU |
| **Elastic GPU** | worker HPA on queue depth (KEDA) | add GPU workers → linear inference throughput |
| **Embarrassingly parallel inference** | tiles are independent | a big raster fans out across workers |
| **Compute-once, serve-many** | artifact registry + 3-tier cache | dashboard reads are O(1); no recompute |
| **Interactive sims** | run on cached in-memory graph | O(query), not O(pipeline) |
| **Durable, versioned artifacts** | object storage keyed by params_hash | unbounded storage, reproducible results, model versioning |
| **Add a city** | scene-scoped key namespace, nothing global mutable | no contention, no migration |

## 5. Background jobs & async processing
- **Celery + Redis** for `infer_scene`, `build_graph`, `run_analytics`. Long work never touches the request thread.
- **Idempotent** by `params_hash` → retries/duplicates are free (cache hits).
- **SSE** streams job progress to the UI (no polling storms).
- **Streaming-ready**: tile results can be emitted as they finish for progressive map rendering (stretch).

## 6. Model versioning & registry
- Checkpoints stored in object storage as `models/toposam_rr/{tag}.pt` with a metadata record (git SHA, `model.yaml`, dataset hash, val metrics).
- API `/version` reports the active checkpoint tag → reproducible, auditable inference.
- Swapping models = changing a config value + rolling the worker Deployment; old artifacts remain valid (keyed by params).

## 7. Object storage layout (MinIO/S3)
```
route-resilience/
  {scene_id}/
    raster/scene.tif
    infer/{params_hash}/road_prob.tif, junction.tif
    graph/{params_hash}/graph.gpkg, graph.json
    analytics/{params_hash}/criticality.json, ...
    sim/{scenario_hash}/result.json
  models/toposam_rr/{tag}.pt
  tiles/{scene_id}/{z}/{x}/{y}.png         # map tiles for the frontend
```

## 8. CI/CD — GitHub Actions
- **CI:** ruff + black + mypy + pytest (backend/ml/graph), Vitest (frontend) on PR.
- **Build:** on main, build+push the three images (api/worker/frontend) to a registry.
- **CD (optional):** `kubectl apply -k deployment/k8s/overlays/{env}` (Kustomize) or Helm.

## 9. Observability
- **structlog** JSON logs with correlation ids (ship to Loki/ELK).
- **Prometheus** metrics (request latency, queue depth, inference time, cache hit rate) + **Grafana** dashboards (stretch). Queue depth is also the worker autoscaling signal → observability and scaling share one metric.

## 10. Resource sizing (rough)
| Service | Demo | Prod (per replica) |
|---------|------|--------------------|
| api | 0.5 CPU / 512MB | 1 CPU / 1GB |
| worker (GPU) | 1 GPU / 8GB VRAM / 8GB RAM | 1 GPU / 16GB |
| redis | 256MB | managed / 2GB |
| minio | 1GB + disk | managed S3 |
| frontend | 256MB | 256MB (static) |

## 11. Hackathon reality check
Bring up **api + worker + redis + minio + frontend** with compose on a single GPU box; the K8s manifests are committed and explained but **not required to run the demo** — they exist to *tell the scalability story* credibly. Judges see real seams (queue, registry, HPA configs), not vaporware.
