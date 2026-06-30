# Training — TopoSAM-RR

This document covers datasets, label generation, the training schedule for a **single mid GPU (T4/3060, 12–16 GB)**, and the all-important **fallback ladder** that guarantees a working model for the demo regardless of how training goes.

> Golden rule: **the demo must never block on training.** The graph/sim/dashboard planes run on a cached `road_prob.tif`. Training improves quality; it is not on the demo's critical path.

---

## 1. Datasets

| Dataset | Use | Notes |
|---------|-----|-------|
| **SpaceNet Roads (AOIs)** | Primary train/val | RGB + road centerline vectors; well-known, license-clear |
| **DeepGlobe Road Extraction** | Augment train | 50cm imagery, binary road masks; large, fast to iterate |
| **Massachusetts Roads** | Quick sanity baseline | Small, classic; good for a 1-hour smoke run |
| **A small held-out city tile** | Demo scene | Reserve one scene the model never trains on → honest demo |

All datasets are read as GeoTIFF (or PNG→geo-wrapped) and processed into the same tile format, so the trainer is dataset-agnostic.

## 2. Label pipeline

```
road centerline vectors (GeoJSON/SHP)
   │ rasterize with buffer (road width ≈ 2–4 px @ 0.5 m GSD)
   ▼
road_mask.tif  (binary)
   │ skeletonize → neighbor-count → ≥3-neighbor pixels
   ▼
junction seeds → 2-D Gaussian (σ=3) → junction_heatmap.tif
```

- **Road target:** rasterized + buffered centerlines (buffer ≈ real road half-width in pixels).
- **Junction target:** auto-derived (see [model.md §4](model.md)). Zero manual annotation.
- Targets are tiled identically to inputs and cached to `data/processed/`.

## 3. Augmentation (the robustness we claim)

| Augmentation | Why |
|--------------|-----|
| Flip / rot90 | Roads are orientation-agnostic; free data |
| Random scale 0.8–1.2 | Multi-resolution robustness |
| Brightness/contrast, CLAHE | Sensor & seasonal variation |
| **Simulated occlusion** (random dark polygons = shadows; coarse dropout = canopy) | **Directly trains occlusion robustness** — the headline capability |
| Cutout over road pixels | Forces the decoder to *inpaint* roads from context |

Occlusion augmentation is non-negotiable: it is the mechanism by which the model learns to recover roads it cannot directly see.

## 4. Training schedule (≈ what fits the 30h window)

```
Phase 0  (0:00–0:30)  Smoke run on Massachusetts: 200 steps, confirm loss ↓, pipeline green
Phase 1  (0:30–3:00)  Main fine-tune: frozen SAM + decoder/heads on SpaceNet+DeepGlobe
                      AdamW lr 3e-4, cosine, warmup 500, AMP, grad-accum → eff. batch 16
Phase 2  (3:00–4:30)  Continue + occlusion-heavy augmentation; checkpoint best by APLS-lite
Phase 3  (optional)   TTA-validated final checkpoint; export ONNX for CPU demo box
```

- **Checkpoint metric = APLS-lite (topology), not pixel IoU.** We care about connectable roads, not pretty pixels. A model with slightly lower IoU but fewer breaks produces a better graph.
- **Memory levers if OOM:** reduce batch to 4 + grad-accum 4; tile 448²; ensure encoder runs under `no_grad`; enable AMP; gradient checkpointing on the decoder only.

## 5. Metrics tracked during training

| Metric | What it tells us |
|--------|------------------|
| Dice / IoU | Pixel overlap (necessary, not sufficient) |
| F1 (precision/recall) | Balance of missed vs hallucinated road |
| **APLS-lite** | Topology fidelity — are shortest paths preserved? (our primary signal) |
| Junction PR / heatmap MSE | Quality of junction seeds for Stage 2/3 |
| Connected-components count vs GT | Proxy for fragmentation (lower break rate = better) |

See [evaluation.md](evaluation.md) for full metric definitions.

## 6. Fallback ladder (guaranteed working model)

| Tier | Model | Trigger | Downstream impact |
|------|-------|---------|-------------------|
| **0** | Pre-baked cached `road_prob.tif` for the demo scene | Always available from hour 1 | None — graph/sim/UI run on it |
| **1** | TopoSAM-RR fine-tuned (frozen SAM) | Default goal | Best quality |
| **2** | Swap encoder → ConvNeXt-T / EfficientNet-b3 | SAM too slow/heavy on the box | Same artifacts, slightly lower quality |
| **3** | Plain U-Net (SMP) | Training instability | Lower quality, identical contract |
| **4** | Classical baseline (threshold + morphology on raw NIR/edges) | Catastrophe | Weak but non-empty graph |

Every tier emits the **same `road_prob.tif` + `junction.tif`**, so the rest of the system is invariant to which tier produced them. **Build the cached-artifact demo path first (Tier 0), then race up the ladder.**

## 7. Reproducibility

- Seeds fixed (`torch`, `numpy`, `random`); `cudnn.deterministic` for eval.
- Every checkpoint stores its `model.yaml` snapshot + git SHA + dataset hash → an artifact-registry entry.
- Training run logged with `structlog` + optional Weights & Biases / TensorBoard (off by default to avoid setup tax).

## 8. Commands (target CLI)

```bash
# Build tiles + labels
python scripts/build_dataset.py --config configs/model.yaml --out data/processed

# Smoke run
python -m ml.toposam_rr.engine.trainer --config configs/model.yaml --max-steps 200

# Full fine-tune
python -m ml.toposam_rr.engine.trainer --config configs/model.yaml

# Inference on a scene → cached artifacts
python scripts/infer_scene.py --raster data/raw/demo_city.tif --ckpt artifacts/models/toposam_rr_best.pt
```
