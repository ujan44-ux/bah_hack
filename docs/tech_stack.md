# Technology Stack

Every choice below is optimized for **one axis: maximum capability per hour of build time**, while staying credible as a production system. Where a "fancier" option exists, we note why we rejected it for a 30-hour build.

## Selection principles

1. **Boring where it doesn't differentiate** — proven libraries for plumbing, novelty only in the pipeline logic.
2. **One language per plane** — Python for ML/graph/backend, TypeScript for frontend. No context-switch tax.
3. **Everything has a CPU fallback** — the demo must run on a laptop if the GPU box dies.
4. **Cache-first** — heavy compute (inference, centrality) is computed once and served from cache.

---

## 1. Languages

| Language | Where | Why | Why not alternative |
|----------|-------|-----|---------------------|
| **Python 3.11** | ML, graph, backend | Ecosystem (PyTorch, NetworkX, rasterio, FastAPI) is unmatched for this domain | Go/Rust faster but no CV/graph ecosystem in time |
| **TypeScript** | Frontend | Type safety on a map-heavy UI prevents whole bug classes | Plain JS — too error-prone for geo data shapes |

## 2. Machine Learning

| Tech | Role | Justification |
|------|------|---------------|
| **PyTorch 2.x** | Model framework | `torch.compile`, mature, SAM weights are native PyTorch |
| **Segment Anything (ViT-B/SAM2 image encoder)** | Pretrained backbone | Strong frozen features → we train only decoder/heads → fits 30h |
| **timm** | Encoder utilities / alt backbones | Drop-in EfficientNet/ConvNeXt fallback if SAM is too heavy |
| **segmentation-models-pytorch** | Reference decoders / fast baseline | Instant U-Net baseline to de-risk while TopoSAM-RR is built |
| **albumentations** | Augmentation | Fast, geo-aware transforms (rotations preserve roads) |
| **kornia** | Differentiable CV ops | Soft-skeleton / boundary ops on GPU for losses |
| **ONNX Runtime** (optional) | Inference portability | Export path for CPU-only demo machines |

**Backbone decision:** SAM image encoder gives occlusion-robust features for free (trained on 1B masks). We *freeze* it and train a lightweight hierarchical decoder + two heads. This is the single biggest time-saver — see [model.md](model.md).

## 3. Geospatial / Raster

| Tech | Role | Why |
|------|------|-----|
| **rasterio** | Read GeoTIFF, CRS, windows | Standard; windowed reads enable tiling without loading full raster |
| **rio-tiler / morecantile** | Tile generation (XYZ/TMS) | Map-ready tiles for the frontend |
| **shapely** | Vector geometry (edges, polygons) | Edge simplification, flood-region intersection |
| **geopandas** | Tabular geo I/O | Export graph as GeoJSON/GeoPackage for GIS users |
| **pyproj** | Projections | Pixel↔geo coordinate conversion |
| **GDAL** (via rasterio wheels) | Backing engine | Already bundled; no separate install pain |

## 4. Image / Topology processing

| Tech | Role |
|------|------|
| **scikit-image** | Skeletonization (`skeletonize`, medial axis), morphology |
| **opencv-python** | Fast thresholding, contour ops, connected components |
| **scipy.ndimage** | Distance transforms, label propagation |
| **numpy** | Array backbone everywhere |

## 5. Graph & Analytics

| Tech | Role | Why |
|------|------|-----|
| **NetworkX** | Graph model + algorithms | Has *everything* (centrality, bridges, articulation, communities) out of the box; correctness over speed |
| **igraph** *(optional accel)* | Heavy centrality on big graphs | C core, ~100× faster betweenness if a city graph is large |
| **scikit-network** *(optional)* | Fast community detection | Louvain at scale |
| **rustworkx** *(stretch)* | Drop-in NetworkX-like, Rust speed | Only if profiling shows NetworkX is the bottleneck |

**Decision:** Build on **NetworkX** for correctness and breadth; swap *only* the hot centrality call to igraph if a graph exceeds ~50k edges. Premature optimization is the enemy at hour 12.

## 6. Backend

| Tech | Role | Why |
|------|------|-----|
| **FastAPI** | REST API | Async, auto OpenAPI docs (free Swagger demo), Pydantic validation |
| **Pydantic v2** | Schemas / config | Validates the messy geo payloads at the boundary |
| **Uvicorn / Gunicorn** | ASGI server | Standard prod serving |
| **Redis** | Cache + job broker + pub/sub | One dependency covers caching, Celery broker, and live updates |
| **Celery** *(or RQ)* | Background jobs | Long inference / centrality run off the request thread |
| **SQLite → PostgreSQL+PostGIS** | Metadata / graph persistence | SQLite for demo, PostGIS-ready for production geo queries |
| **structlog** | Structured logging | JSON logs, correlation IDs |

**REST vs alternatives:** REST/FastAPI over GraphQL (no time for schema overhead) and over gRPC (frontend needs plain HTTP/JSON). See [backend.md](backend.md).

## 7. Frontend

| Tech | Role | Why |
|------|------|-----|
| **Next.js (React 18, App Router)** | App shell | SSR-ready, file routing, fast DX |
| **TypeScript** | Safety | Geo shapes are error-prone |
| **MapLibre GL JS** | Base map | Open-source (no Mapbox token), vector tiles, GPU rendering |
| **deck.gl** | Data overlays | GPU-accelerated rendering of 100k+ edges, heatmaps, path layers |
| **Zustand** | State management | Minimal boilerplate vs Redux; perfect for map + sim state |
| **TanStack Query** | Server state / caching | Dedupes API calls, background refetch for live metrics |
| **Tailwind CSS + shadcn/ui** | Styling / components | Production-looking dashboard fast |
| **Recharts** | Metric charts | Resilience curves, centrality histograms |

**Map stack decision:** MapLibre (base) + deck.gl (overlays) beats Leaflet for large vector data — Leaflet's DOM rendering chokes past a few thousand features; deck.gl renders on the GPU.

## 8. Deployment & Infra

| Tech | Role | Why |
|------|------|-----|
| **Docker + docker-compose** | Local & demo orchestration | One command brings up api+worker+redis+frontend |
| **Kubernetes manifests (provided, not required)** | Scale story | Shows production-readiness; HPA on the worker |
| **MinIO (S3-compatible)** | Object storage | Tiles, checkpoints, graph artifacts; swappable for AWS S3 |
| **NGINX** | Reverse proxy / static | TLS + routing in front of API and frontend |
| **GitHub Actions** | CI | Lint + test + build images |

## 9. Quality / Tooling

| Tech | Role |
|------|------|
| **pytest** | Backend/ML/graph tests |
| **ruff + black** | Lint + format (one tool, fast) |
| **mypy** | Type checking on services |
| **Vitest + Playwright** | Frontend unit + e2e |
| **pre-commit** | Gate before commit |
| **Prometheus + Grafana** *(stretch)* | Metrics dashboards |

## 10. Stack at a glance

```
ML:        PyTorch · SAM encoder · timm · albumentations · kornia
Geo:       rasterio · rio-tiler · shapely · geopandas · pyproj
Topology:  scikit-image · opencv · scipy
Graph:     NetworkX (+igraph accel)
Backend:   FastAPI · Pydantic · Redis · Celery · PostGIS-ready
Frontend:  Next.js · MapLibre · deck.gl · Zustand · TanStack Query · Tailwind
Infra:     Docker · K8s · MinIO · NGINX · GitHub Actions
```

## 11. What we deliberately avoided

| Tempting | Rejected because |
|----------|------------------|
| Training SAM from scratch | Weeks of compute; we freeze + fine-tune heads |
| GraphQL | Schema overhead with no payoff for this UI |
| Custom CUDA kernels | Zero ROI in 30h; PyTorch + igraph suffice |
| Microservices everywhere | Modular monolith is faster to build and debug (see backend.md) |
| Mapbox GL | Token/licensing friction; MapLibre is the open fork |
| Cesium / 3D globe | Visual flash, but 2D map answers the planning question better |
