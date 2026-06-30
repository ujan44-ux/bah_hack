# API Reference

REST API served by FastAPI. All responses JSON unless noted; geo payloads use **GeoJSON** (WGS84 / EPSG:4326). Interactive docs auto-generated at `/docs` (Swagger) and `/redoc`. Base path: `/api/v1`.

Conventions: heavy operations return **`202 Accepted` + `job_id`** and run async; results are fetched once the job completes. Errors use RFC-7807 problem-details with a `correlation_id`.

---

## 1. Scenes

### `POST /scenes`
Upload or register a satellite raster.
```
multipart: file=<GeoTIFF>   OR   json: { "uri": "s3://.../scene.tif" }
→ 201 { "scene_id": "scn_ab12", "bounds": [...], "crs": "EPSG:32643", "size_px": [H,W] }
```

### `GET /scenes` / `GET /scenes/{scene_id}`
List scenes / fetch one scene's metadata + available artifacts + processing status.

### `POST /scenes/{scene_id}/process`
Run the full pipeline (infer → graph → analytics) async.
```
json (optional): { "params_override": { "binarize.high": 0.6 } }
→ 202 { "job_id": "job_77x", "stages": ["infer","graph","analytics"] }
```

---

## 2. Inference (Stage 1)

### `POST /scenes/{scene_id}/infer`
Run only segmentation. `→ 202 { job_id }`.

### `GET /scenes/{scene_id}/maps/{kind}`
Fetch a probability map. `kind ∈ {road_prob, junction}`.
```
?format=png|tif|tilejson    (tilejson → XYZ tile endpoint for the map)
→ 200 image / 200 { tilejson, tiles:[".../{z}/{x}/{y}.png"] }
```

---

## 3. Graph (Stages 2–4)

### `GET /scenes/{scene_id}/graph`
Fetch the attributed road graph.
```
?format=geojson|json|gpkg   &  ?include_healed=true
→ 200 {
  "nodes": [{"id":12,"lonlat":[...],"degree":3,"junction_conf":0.81}],
  "edges": [{"id":5,"u":12,"v":47,"geometry":{...},
             "length_m":44.0,"direction":92.0,"curvature":0.07,
             "confidence":0.66,"healed":false,"travel_cost":3.9}]
}
```

### `GET /scenes/{scene_id}/graph/stats`
Summary: node/edge counts, total length, #healed edges, #components (pre/post heal).

---

## 4. Analytics (Stage 5)

### `GET /scenes/{scene_id}/analytics/criticality`
Per-edge & per-node criticality + components.
```
?normalize=true
→ 200 {
  "edges": [{"id":5,"betweenness":0.42,"is_bridge":true,"criticality":0.88}],
  "nodes": [{"id":12,"betweenness":0.39,"is_articulation":true,"criticality":0.91}],
  "communities": [{"id":0,"node_ids":[...]}],
  "summary": {"n_bridges":14,"n_articulation":9,"n_communities":6}
}
```

### `GET /scenes/{scene_id}/analytics/{layer}`
Fetch a single layer. `layer ∈ {betweenness, edge_betweenness, bridges, articulation, communities}`.

---

## 5. Simulation (Stage 6)

### `POST /scenes/{scene_id}/simulate`
Run a stress-test scenario on the cached graph (fast, usually synchronous).
```
json: {
  "type": "flooding",                  // node_failure | road_closure | flooding | congestion | targeted | random
  "params": { "polygon": {...GeoJSON...}, "depth_m": 1.2 },
  "reroute": [ {"od":[12,455]} ],      // optional O-D pairs to re-route
  "compare_to": "baseline"
}
→ 200 SimulationResult {
  "metrics": {"resilience_index":0.62,"efficiency_before":0.041,"efficiency_after":0.025,
              "avg_path_len_before":1820,"avg_path_len_after":2410,"travel_delay":590,
              "components_before":1,"components_after":3,"lcc_fraction":0.78,"lost_demand":0.14},
  "affected": {"nodes":[...],"edges":[...]},
  "reroutes": [{"od":[12,455],"before":{...},"after":{...},"delay":380}]
}
```

### `POST /scenes/{scene_id}/simulate/curve`
Resilience curve: iteratively remove top-`k` elements by a strategy.
```
json: { "strategy": "targeted_betweenness", "k": 25 }   // or "random", "targeted_bridge"
→ 200 { "k": [0..25], "efficiency": [...], "lcc_fraction": [...], "auc": 0.71 }
```

### `POST /scenes/{scene_id}/route`
Shortest path between two points (snaps to nearest nodes).
```
json: { "from":[lon,lat], "to":[lon,lat], "scenario": {optional} }
→ 200 { "path":{...GeoJSON LineString...}, "cost": 412.0, "nodes":[...] }
```

---

## 6. Jobs

### `GET /jobs/{job_id}`
```
→ 200 { "job_id":"job_77x","status":"running","progress":0.6,
        "current_stage":"graph","stages_done":["infer"],"error":null }
```

### `GET /jobs/{job_id}/stream`  (Server-Sent Events)
Live progress events: `{stage, progress, message}` until `status=done|failed`. Drives the dashboard's live progress without polling.

---

## 7. Health & meta

| Endpoint | Purpose |
|----------|---------|
| `GET /health` | liveness |
| `GET /ready` | readiness (model loaded, redis/storage reachable) |
| `GET /version` | git SHA, model checkpoint tag |
| `GET /docs` `/redoc` | auto OpenAPI docs |

---

## 8. Error format (RFC-7807)
```json
{
  "type": "https://route-resilience/errors/graph-build",
  "title": "GraphBuildError",
  "status": 422,
  "detail": "Skeletonization produced an empty graph (mask < min_object_px).",
  "correlation_id": "c-9f3a1",
  "scene_id": "scn_ab12"
}
```

## 9. Status codes
`200` ok · `201` created · `202` accepted (async job) · `400` bad request · `404` not found · `409` conflict (job already running) · `422` pipeline/domain error · `500` unexpected.

## 10. Versioning
Path-versioned (`/api/v1`). Artifact-producing endpoints echo the `params_hash` so clients can detect when results came from changed parameters.
