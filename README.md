# bah_hack
# Route Resilience
### Occlusion-Robust Road Extraction & Graph-Theoretic Criticality Analysis for Urban Mobility

> ISRO NNRMS Hackathon — Problem Statement 4

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Repository Structure](#repository-structure)
4. [Technology Stack](#technology-stack)
5. [Phase I — Segmentation Service (Python)](#phase-i--segmentation-service-python)
6. [Phase II — Graph Healing Service (Python)](#phase-ii--graph-healing-service-python)
7. [Phase III — Network Analysis Service (Python)](#phase-iii--network-analysis-service-python)
8. [Phase IV — Backend API (Java / Spring Boot)](#phase-iv--backend-api-java--spring-boot)
9. [Phase IV — Frontend Dashboard (React + Leaflet.js)](#phase-iv--frontend-dashboard-react--leafletjs)
10. [Quick Start](#quick-start)
11. [Dataset Setup](#dataset-setup)
12. [Evaluation Metrics](#evaluation-metrics)
13. [Parallel Workflow (30-hour Hackathon)](#parallel-workflow-30-hour-hackathon)
14. [API Reference](#api-reference)
15. [Configuration](#configuration)

---

## Project Overview

Modern satellite imagery of Indian metropolises (e.g., Bengaluru) suffers from **spectral blindness** — tree canopies, building shadows, vehicles, and cloud cover break road segmentation masks into disconnected fragments, making them useless for real-world routing.

This project delivers an end-to-end pipeline that:

- **Sees through occlusions** using a context-aware Transformer + U-Net++ segmentation model
- **Heals topological gaps** using Minimum Spanning Tree (MST) and Disjoint Set algorithms
- **Identifies bottlenecks** via Betweenness Centrality on the reconstructed road graph
- **Simulates collapse** by ablating high-centrality nodes to produce a Resilience Index
- **Visualises everything** on an interactive Leaflet.js map with rerouting simulation

---

## Architecture

```
Satellite imagery (Sentinel-2 / LISS-IV / Cartosat-3)
        │
        ▼
┌─────────────────────────────────────┐
│  Phase I  │  Python · PyTorch       │
│           │  UNet++ / Transformer   │  → Binary road mask (GeoTIFF)
│           │  Albumentations · GDAL  │
└───────────────────────┬─────────────┘
                        │
                        ▼
┌─────────────────────────────────────┐
│  Phase II │  Python · NetworkX      │
│           │  scikit-image · OSMnx   │  → Weighted road graph (GeoJSON)
│           │  MST · Disjoint Sets    │
└───────────────────────┬─────────────┘
                        │
                        ▼
┌─────────────────────────────────────┐
│  Phase III│  Python · NetworkX/PyG  │
│           │  Betweenness centrality │  → Criticality scores + Resilience Index
│           │  Node ablation          │
└───────────────────────┬─────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────────┐
│  Phase IV │  Java 17 · Spring Boot REST API               │
│           │  React + Leaflet.js  ·  Streamlit (fallback)  │
│           │  PostGIS (optional)  ·  GeoJSON file store     │
└───────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
route-resilience/
├── python/                         # All Python services
│   ├── segmentation/
│   │   ├── data/                   # Dataset loaders, augmentation
│   │   │   ├── dataset.py
│   │   │   ├── augment.py          # Albumentations pipeline
│   │   │   └── occlusion_sim.py    # Shadow / canopy / cloud simulation
│   │   ├── models/
│   │   │   ├── unetpp.py           # UNet++ with ResNet backbone
│   │   │   ├── transformer_seg.py  # Swin-T / SegFormer wrapper
│   │   │   └── loss.py             # Dice + IoU + boundary loss
│   │   ├── train.py
│   │   ├── evaluate.py
│   │   └── infer.py                # Produces mask GeoTIFF
│   │
│   ├── graph/
│   │   ├── skeletonize.py          # Morphological thinning → centrelines
│   │   ├── node_extraction.py      # Intersection + endpoint detection
│   │   ├── healing.py              # MST + Disjoint Set gap bridging
│   │   ├── export.py               # NetworkX → GeoJSON / pickle
│   │   └── evaluate_graph.py       # OSM benchmark comparison
│   │
│   ├── analysis/
│   │   ├── centrality.py           # Betweenness centrality
│   │   ├── ablation.py             # Node removal simulation
│   │   ├── resilience.py           # Resilience Index computation
│   │   └── scenarios.py            # Flood / accident / construction presets
│   │
│   ├── dashboard/
│   │   └── streamlit_app.py        # Streamlit fallback UI
│   │
│   ├── requirements.txt
│   └── config.yaml
│
├── java/                           # Spring Boot REST API
│   ├── pom.xml
│   └── src/main/java/com/isro/routeresilience/
│       ├── RouteResilienceApplication.java
│       ├── controller/
│       │   ├── GraphController.java
│       │   ├── AblationController.java
│       │   └── HeatmapController.java
│       ├── service/
│       │   ├── GraphService.java
│       │   ├── AblationService.java
│       │   └── ResilienceService.java
│       ├── model/
│       │   ├── RoadGraph.java
│       │   ├── NodeCriticality.java
│       │   └── AblationResult.java
│       └── config/
│           └── CorsConfig.java
│
├── frontend/                       # React + Leaflet.js
│   ├── package.json
│   ├── public/
│   └── src/
│       ├── App.jsx
│       ├── components/
│       │   ├── MapView.jsx          # Leaflet map container
│       │   ├── HeatmapLayer.jsx     # Criticality colour overlay
│       │   ├── NodePopup.jsx        # Click-to-ablate UI
│       │   ├── ReroutePanel.jsx     # Before/after path comparison
│       │   └── ResilienceGauge.jsx  # Index display
│       ├── hooks/
│       │   └── useGraph.js          # REST calls to Spring Boot
│       └── utils/
│           └── colorScale.js        # Criticality → colour mapping
│
├── data/                            # Not committed — see Dataset Setup
│   ├── raw/
│   ├── processed/
│   ├── masks/
│   └── graphs/
│
├── docker-compose.yml
└── README.md
```

---

## Technology Stack

| Layer | Language | Key Libraries / Frameworks |
|---|---|---|
| Data preprocessing | Python 3.10+ | GDAL, Rasterio, OpenCV, NumPy |
| Data augmentation | Python | Albumentations |
| Segmentation model | Python | PyTorch 2.x, segmentation-models-pytorch, timm |
| Skeletonization | Python | scikit-image, FilFinder, OSMnx |
| Graph processing | Python | NetworkX 3.x, PyTorch Geometric (optional) |
| REST API | Java 17 | Spring Boot 3.x, Spring Web, Jackson |
| Build tool | Java | Maven 3.9+ |
| Frontend map | JavaScript | React 18, Leaflet.js, Axios |
| Frontend bundler | JavaScript | Vite |
| Rapid prototype UI | Python | Streamlit |
| Geospatial storage | — | GeoJSON files, PostGIS (optional) |
| Visualisation | Python | Matplotlib, QGIS (offline review) |

---

## Phase I — Segmentation Service (Python)

### Setup

```bash
cd python
pip install -r requirements.txt
```

### Training

```bash
python segmentation/train.py \
  --config config.yaml \
  --model unetpp \
  --backbone resnet50 \
  --epochs 50 \
  --batch-size 8 \
  --loss combined        # dice + iou + boundary
```

### Model options (`--model`)

- `unetpp` — UNet++ with ResNet/EfficientNet backbone (baseline, fast to train)
- `deeplabv3plus` — DeepLabV3+ (strong on dense urban)
- `transformer` — Swin-T encoder + UPerNet decoder (best long-range context)

### Inference (produces GeoTIFF mask)

```bash
python segmentation/infer.py \
  --input data/raw/tile_001.tif \
  --model-checkpoint checkpoints/best.pth \
  --output data/masks/tile_001_mask.tif
```

### Occlusion simulation

`data/occlusion_sim.py` synthetically adds shadows, canopy patches, and cloud streaks to training tiles so the model learns to infer road continuity under cover.

### Loss function

```python
# loss.py
total_loss = (
    0.4 * dice_loss(pred, target)
  + 0.4 * iou_loss(pred, target)
  + 0.2 * boundary_loss(pred, target)
)
```

---

## Phase II — Graph Healing Service (Python)

### Run (takes a mask GeoTIFF, writes a GeoJSON graph)

```bash
python graph/export.py \
  --mask data/masks/tile_001_mask.tif \
  --output data/graphs/tile_001_graph.geojson
```

### Pipeline internals

1. **Skeletonize** (`skeletonize.py`) — morphological thinning to 1-pixel centrelines using `skimage.morphology.skeletonize`
2. **Node extraction** (`node_extraction.py`) — intersections (3+ neighbours) and endpoints (1 neighbour) become graph nodes; segments become edges weighted by Euclidean length
3. **Gap detection** (`healing.py`) — identifies dangling endpoints within a configurable Euclidean threshold (default 20 pixels)
4. **MST healing** — builds candidate connections between gap-endpoints and selects the Minimum Spanning Tree subset that maximises connectivity while respecting angular alignment constraints
5. **Disjoint Set merge** — Union-Find tracks connected components; healing terminates when the largest component reaches ≥ 95% of all nodes (configurable)

### Evaluate against OSM

```bash
python graph/evaluate_graph.py \
  --predicted data/graphs/tile_001_graph.geojson \
  --osm-bbox "12.97,77.59,13.01,77.63" \
  --metric path-length-error
```

---

## Phase III — Network Analysis Service (Python)

### Run full analysis

```bash
python analysis/centrality.py \
  --graph data/graphs/tile_001_graph.geojson \
  --output data/graphs/tile_001_centrality.json

python analysis/ablation.py \
  --graph data/graphs/tile_001_graph.geojson \
  --centrality data/graphs/tile_001_centrality.json \
  --top-n 10 \
  --output data/graphs/tile_001_ablation.json
```

### Resilience Index

```
R = mean_shortest_path(baseline) / mean_shortest_path(perturbed)
```

A value of R < 0.6 indicates a highly vulnerable network. Results are written as JSON consumed by the Spring Boot API.

### Disaster scenario presets

```bash
python analysis/scenarios.py --scenario flood --zone "12.98,77.60,12.99,77.61"
python analysis/scenarios.py --scenario accident --node-id 4821
python analysis/scenarios.py --scenario construction --edge-ids 1042,1043,1044
```

---

## Phase IV — Backend API (Java / Spring Boot)

### Build & run

```bash
cd java
mvn clean package -DskipTests
java -jar target/route-resilience-0.0.1-SNAPSHOT.jar
# Server starts on http://localhost:8080
```

### Key endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/graph` | Full road graph as GeoJSON |
| `GET` | `/api/heatmap` | Criticality scores for all nodes |
| `POST` | `/api/ablate` | Remove node(s) and return updated path lengths |
| `GET` | `/api/resilience` | Current Resilience Index |
| `GET` | `/api/scenarios` | List available disaster presets |
| `POST` | `/api/scenarios/{name}` | Apply a preset scenario |

### Example: ablate a node

```bash
curl -X POST http://localhost:8080/api/ablate \
  -H "Content-Type: application/json" \
  -d '{"nodeIds": [4821, 3302]}'
```

Response:

```json
{
  "resilienceIndex": 0.54,
  "avgPathLengthBaseline": 1420.3,
  "avgPathLengthPerturbed": 2631.7,
  "affectedRoutes": 142,
  "reroutedPaths": [...]
}
```

### Project structure (`java/src/`)

- `controller/` — thin REST layer, delegates to services
- `service/` — reads GeoJSON/JSON files produced by Phase II–III Python scripts; performs in-memory graph queries using JGraphT (lightweight, no NetworkX dependency in Java)
- `model/` — Jackson-serialised POJOs for all API responses
- `config/CorsConfig.java` — enables CORS for the React frontend

---

## Phase IV — Frontend Dashboard (React + Leaflet.js)

### Setup & run

```bash
cd frontend
npm install
npm run dev
# Opens at http://localhost:5173
```

### Key components

**`MapView.jsx`** — renders the base OSM tile layer and loads the road graph GeoJSON as a Leaflet GeoJSON layer. Road segments are coloured by criticality score using a red-yellow-green scale.

**`HeatmapLayer.jsx`** — overlays node circles sized and coloured by Betweenness Centrality. Nodes above the 90th-percentile threshold are pulsed in red to highlight "gatekeeper" bottlenecks.

**`NodePopup.jsx`** — clicking any node shows its centrality score and a "Simulate failure" button. On click, the component calls `POST /api/ablate` and triggers a re-render of affected paths.

**`ReroutePanel.jsx`** — side panel showing before/after average path length, the Resilience Index gauge, and a list of the top 5 most-affected routes with estimated travel time increase.

### Build for production

```bash
npm run build
# Output in frontend/dist/ — serve with any static host or embed in Spring Boot /static/
```

---

## Quick Start

### With Docker Compose (recommended)

```bash
# 1. Add your tiles to data/raw/
# 2. Build and start all services
docker-compose up --build

# Services:
#   Python pipeline:  triggered via REST or CLI
#   Spring Boot API:  http://localhost:8080
#   React frontend:   http://localhost:5173
#   Streamlit:        http://localhost:8501
```

### Manual (no Docker)

```bash
# Terminal 1 — run Python pipeline end-to-end on a sample tile
cd python
pip install -r requirements.txt
python segmentation/infer.py --input data/raw/sample.tif --output data/masks/sample_mask.tif
python graph/export.py --mask data/masks/sample_mask.tif --output data/graphs/sample.geojson
python analysis/centrality.py --graph data/graphs/sample.geojson --output data/graphs/sample_centrality.json
python analysis/ablation.py --graph data/graphs/sample.geojson --centrality data/graphs/sample_centrality.json --output data/graphs/sample_ablation.json

# Terminal 2 — Spring Boot API
cd java && mvn spring-boot:run

# Terminal 3 — React frontend
cd frontend && npm install && npm run dev

# Optional — Streamlit prototype (no Java/React needed)
cd python && streamlit run dashboard/streamlit_app.py
```

---

## Dataset Setup

```bash
mkdir -p data/raw data/processed data/masks data/graphs
```

| Dataset | How to obtain | Place in |
|---|---|---|
| SpaceNet Roads | https://spacenet.ai/roads/ | `data/raw/spacenet/` |
| DeepGlobe Roads | https://competitions.codalab.org/competitions/18467 | `data/raw/deepglobe/` |
| OpenSatMap | https://opensatmap.github.io | `data/raw/opensatmap/` |
| Sentinel-2 | https://scihub.copernicus.eu (free) | `data/raw/sentinel2/` |
| Resourcesat LISS-IV | NRSC BHUVAN portal | `data/raw/lissiv/` |
| Cartosat-3 | Provided by ISRO during hackathon | `data/raw/cartosat3/` |
| OSM vector masks | Auto-generated (see below) | `data/processed/osm_masks/` |

### Auto-generate OSM ground-truth masks

```bash
# Uses OSMnx to download road vectors and rasterise to PNG masks
python graph/osm_mask_gen.py --bbox "12.97,77.59,13.01,77.63" --resolution 5.8
```

---

## Evaluation Metrics

| Metric | Target | Measured by |
|---|---|---|
| IoU (overall) | ≥ 0.70 | `segmentation/evaluate.py` |
| Dice score | ≥ 0.80 | `segmentation/evaluate.py` |
| Occlusion-Recall | ≥ 0.60 | `segmentation/evaluate.py --occluded-only` |
| Connectivity Ratio | ≥ +30 pp after healing | `graph/evaluate_graph.py` |
| Relaxed IoU (3 px buffer) | ≥ 0.85 | `graph/evaluate_graph.py --buffer 3` |
| Path Length Error vs OSM | ≤ 15% | `graph/evaluate_graph.py --metric path-length-error` |
| Generalisation (3 terrains) | Reported per terrain | `segmentation/evaluate.py --terrain` |

---

## Parallel Workflow (30-hour Hackathon)

```
Hour  0–2   | Both sub-teams: dataset download, env setup, Docker verify
Hour  2–14  | Sub-team A: train segmentation model (GPU)
            | Sub-team B: build graph healing + analysis on mock/OSM vectors
Hour 14–20  | A → produces first real masks; B integrates masks into graph pipeline
Hour 20–24  | Spring Boot API scaffolded; endpoints wired to Phase II–III outputs
Hour 24–28  | React frontend: map, heatmap layer, ablation popup
Hour 28–30  | Integration testing, demo recording, README polish
```

Sub-team B can start immediately with OSM road vectors as mock input — no GPU or trained model is needed to develop and test the entire graph pipeline.

---

## API Reference

Full OpenAPI spec is served at `http://localhost:8080/swagger-ui.html` when the Spring Boot server is running (Springdoc OpenAPI is included in `pom.xml`).

---

## Configuration

### `python/config.yaml`

```yaml
segmentation:
  tile_size: 512
  overlap: 64
  model: transformer          # unetpp | deeplabv3plus | transformer
  backbone: resnet50
  num_epochs: 50
  learning_rate: 0.0001
  loss_weights: [0.4, 0.4, 0.2]  # dice, iou, boundary

graph:
  gap_threshold_px: 20        # max pixel gap to attempt healing
  angular_tolerance_deg: 30   # max angle deviation for healed edges
  connectivity_target: 0.95   # stop healing when this fraction connected

analysis:
  top_n_nodes: 20             # ablate top-N by betweenness centrality
  resilience_sample_pairs: 500  # random node pairs for path-length estimation

api:
  graph_path: data/graphs/tile_001_graph.geojson
  centrality_path: data/graphs/tile_001_centrality.json
  ablation_path: data/graphs/tile_001_ablation.json
```

### `java/src/main/resources/application.properties`

```properties
server.port=8080
data.graph-path=../data/graphs/tile_001_graph.geojson
data.centrality-path=../data/graphs/tile_001_centrality.json
data.ablation-path=../data/graphs/tile_001_ablation.json
spring.web.cors.allowed-origins=http://localhost:5173
```

---

## Acknowledgements

Built for ISRO NNRMS Hackathon (PS-4) · Datasets: SpaceNet, DeepGlobe, OpenSatMap, Copernicus Sentinel, NRSC BHUVAN · Ground truth: OpenStreetMap contributors
