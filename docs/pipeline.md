# Pipeline — Stage 1 (Segmentation) & Stage 2 (Topology Extraction)

This document covers the path from **raw raster** to a **raw road graph**. Stage 1's model internals live in [model.md](model.md); here we focus on the *pipeline* — preprocessing, tiling, binarization, skeletonization, and vectorization — i.e. how probability maps become nodes and edges.

---

## Stage 1 — Occlusion-aware Segmentation (pipeline view)

```
Satellite GeoTIFF
   │
   ▼ ┌──────────────────────────────────────────────┐
     │ 1. Raster reader (rasterio)                   │  read bands, CRS, transform
     │ 2. Tiler (512², overlap 64, reflect-pad)      │  windowed reads, no full load
     │ 3. Normalizer (SAM mean/std) → tensor         │  [B,3,512,512]
     │ 4. TopoSAM-RR forward (AMP, no_grad)          │  → per-tile prob/junction
     │ 5. Stitcher (cosine seam-blend)               │  reassemble full scene
     └──────────────────────────────────────────────┘
   ▼
road_prob.tif  [H,W] ∈ [0,1]     junction.tif  [H,W] ∈ [0,1]   (georeferenced)
```

**Module map (`ml/.../inference/`):** `predictor.py` (tile loop + batching), `stitch.py` (seam blend + geo-write).

**Why overlap + seam blend?** A tile boundary that drops a few road pixels becomes a *graph break* downstream — far more costly than a pixel error. Overlapping tiles and cosine feathering guarantee continuity across seams.

---

## Stage 2 — Topology Extraction

Goal: turn the continuous probability maps into a **graph of nodes and edges** with positions, confidences, and junction awareness — the substrate the healing stage repairs.

```
road_prob.tif + junction.tif
   │
   ▼ ┌─────────────────────────────────────────────────────────┐
     │ A. Binarize        hysteresis threshold + morphology      │
     │ B. Skeletonize     Zhang–Suen / medial axis               │
     │ C. Refine          junction-guided seeding + spur prune    │
     │ D. Node extract    endpoints + junctions = graph nodes     │
     │ E. Edge extract    trace skeleton paths between nodes       │
     │ F. Vectorize       simplify polylines, attach confidence    │
     └─────────────────────────────────────────────────────────┘
   ▼
graph_raw  (nodes, edges, possibly disconnected)   → Stage 3 healing
```

### A. Binarization — `graph/extraction/binarize.py`

We use **hysteresis (double) thresholding**, not a single cutoff:

```
high = 0.55, low = 0.30
strong = road_prob ≥ high          (definitely road)
weak   = road_prob ≥ low           (maybe road)
keep weak pixels only if connected to a strong pixel  (flood fill / reconstruction)
```

Then morphology: closing (kernel 3) to bridge 1–2 px gaps, remove objects < `min_object_px`.

**Why hysteresis?** A single threshold either drops faint (occluded) roads or floods noise. Hysteresis keeps faint road *that is continuous with confident road* — exactly the occlusion-recovery behavior we want. Math: [math.md#hysteresis-thresholding](math.md).

### B. Skeletonization — `graph/extraction/skeletonize.py`

Reduce the binary road blob to a **1-pixel-wide centerline**:
- **Zhang–Suen thinning** (default): iterative boundary peeling preserving connectivity.
- **Medial axis** (alt): gives a distance map too (→ road width estimate).

Output: 1-px skeleton + (optional) per-pixel distance-to-edge (road half-width). Math: [math.md#skeletonization](math.md).

### C. Junction-guided refinement — `graph/extraction/skeletonize.py`

Raw skeletons are messy at intersections (spurs, staircases, false branches). We use the **junction heatmap** from Stage 1 to clean them:

```
1. Detect skeleton junctions: pixels with ≥3 skeleton neighbors (neighbor-count conv).
2. Snap them to local maxima of junction.tif within a small radius
   → corrects off-by-a-few-pixels intersection placement.
3. Prune spurs: dangling branches shorter than prune_spur_len that don't reach a junction.
4. Where the model predicts a strong junction but the skeleton has a gap,
   mark a 'healing hint' (helps Stage 3 prioritize that reconnection).
```

**Why use the junction head here?** Pure morphological skeletons get intersections wrong, which corrupts node placement and edge degree. The learned junction prior gives sub-skeleton accuracy at exactly the points that matter most for routing.

### D & E. Node & edge extraction — `graph/extraction/vectorize.py`

```
Nodes  = skeleton endpoints (1 neighbor)  ∪  junctions (≥3 neighbors)
Edges  = maximal skeleton paths connecting two nodes (degree-2 chains between nodes)
```

- Walk the skeleton from each node along degree-2 pixels until the next node → one edge (a polyline of pixel coords).
- Each edge stores its **pixel polyline**; each node stores its pixel coord + (via the geo-transform) its real-world coordinate.

### F. Vectorize & annotate — `graph/extraction/vectorize.py`

- **Simplify** each edge polyline with Douglas–Peucker (tol `simplify_tol`) → fewer vertices, smoother geometry, lighter payloads.
- Drop edges shorter than `min_edge_len` (skeleton noise).
- Attach **segmentation confidence** to each edge = mean `road_prob` sampled along its polyline; attach **junction confidence** to each node = `junction.tif` value at its location.
- Convert pixel coords → geo coords via the raster transform (pyproj/rasterio).

**Output object — `graph_raw`** (written as GeoPackage + JSON):
```json
{
  "nodes": [{"id": 12, "xy_px": [340, 512], "lonlat": [...], "junction_conf": 0.81, "degree": 3}],
  "edges": [{"u": 12, "v": 47, "polyline_px": [[...]], "seg_conf": 0.66, "length_px": 88}]
}
```

This graph is typically **disconnected** (occlusions left gaps). Reconnecting it is Stage 3's job → [graph_engine.md](graph_engine.md).

---

## Degradation & robustness

| Failure | Fallback |
|---------|----------|
| Junction head weak/absent | Skip junction-snap; use morphological junctions only |
| Skeleton too noisy | Increase closing kernel + spur prune; raise `low` threshold |
| Scene huge | Stage 2 runs tile-wise on the stitched mask in blocks, then merges node/edge sets at block borders |

## Module map

```
graph/extraction/
├── binarize.py      # hysteresis + morphology
├── skeletonize.py   # thinning + junction-guided refinement + spur prune
└── vectorize.py     # node/edge extraction, simplify, confidence sampling, geo-projection
```
