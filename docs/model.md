# TopoSAM-RR — Model Architecture

**TopoSAM-RR** (Topology-aware Segment-Anything for Route Resilience) is the deep model of Stage 1. It is **not** a reproduction of SAMRoad++. It is an engineered, hackathon-realistic variant built on one decision:

> **Freeze the SAM image encoder; train only a lightweight hierarchical decoder and two task heads.**

This buys occlusion-robust features (SAM was trained on 1B masks across enormous visual diversity) for *zero* training cost, and keeps the trainable parameter count small enough to converge on a **single mid GPU (T4 / RTX 3060, 12–16 GB)** within a few hours.

---

## 1. Why this design

| Requirement | Design response |
|-------------|-----------------|
| Roads hidden under canopy/shadow | Strong, occlusion-robust frozen SAM features + a decoder that fuses multi-scale context |
| Connectable topology, not just pixels | A dedicated **Junction head** + skeleton-aware losses so the graph stage gets seeds |
| Train in hours on one GPU | Freeze encoder (~90M params frozen), train decoder+heads (~5–10M params) |
| Confidence for downstream healing | Heads emit calibrated probability maps used as `seg_confidence` / `junction_confidence` in Stage 3 |
| Big rasters | Tiled inference with overlap + seam blending |

**Rejected alternatives:** training a backbone from scratch (weeks of compute); HRNet/DeepLab end-to-end (more params to train, weaker occlusion priors than SAM); pure U-Net (fine as a *baseline/fallback* — we keep it in `ml/` to de-risk, see §9).

---

## 2. Top-level tensor flow

```
Input tile            x:  [B, 3, 512, 512]
        │  Preprocess (normalize to SAM stats, pad to /16)
        ▼
SAM Image Encoder (FROZEN, ViT-B)
        │  patch embed (16×16) → transformer
        ├── neck feature        f16: [B, 256, 32, 32]      (1/16 scale)
        │
        │  (we also tap intermediate ViT blocks for pseudo-multi-scale)
        ├── early-block proj     f8: [B, 128, 64, 64]      (1/8,  via upsample-proj)
        └── earlier-block proj   f4: [B,  64, 128, 128]    (1/4,  via upsample-proj)
        ▼
Hierarchical Multi-Scale Decoder (TRAINABLE)
        │  FPN-style top-down fusion + lightweight attention gates
        ▼
Shared decoder feature  d:  [B, 64, 128, 128]  → upsample → [B, 64, 512, 512]
        │
        ├──► Road Segmentation Head  → road_logits     [B, 1, 512, 512]
        └──► Junction Prediction Head → junction_logits [B, 1, 512, 512]
        ▼
Sigmoid → road_prob [B,1,512,512], junction_heat [B,1,512,512]
```

> **Note on multi-scale from a ViT:** SAM's ViT is single-scale (1/16). We synthesize a feature pyramid by tapping **intermediate transformer blocks** (e.g. blocks 3/6/9/12) and projecting+upsampling them to 1/8 and 1/4. This "ViT-FPN" trick gives the decoder the spatial detail it needs to recover thin roads without any encoder fine-tuning. (If using SAM2, its hierarchical image encoder already yields native multi-scale — drop the projection trick.)

---

## 3. Component detail

### 3.1 Preprocessing
- Resize/pad tile to a multiple of 16 (SAM patch size).
- Normalize with SAM's pixel mean/std (`[123.675, 116.28, 103.53] / [58.395, 57.12, 57.375]`).
- Output: `[B, 3, 512, 512]`.

### 3.2 Encoder — SAM ViT-B (frozen)
- Patch embedding 16×16 → 32×32 tokens of dim 768.
- 12 transformer blocks with windowed attention + a few global-attention blocks.
- Neck: 1×1 conv → 3×3 conv → `f16 [B, 256, 32, 32]`.
- **All parameters frozen** (`requires_grad=False`). Runs under `torch.no_grad()` at train time → less memory, faster.
- Intermediate taps: hook outputs of selected blocks, reshape tokens → `[B, 768, 32, 32]`, project to channels and bilinear-upsample to build `f8`, `f4`.

### 3.3 Decoder — Hierarchical Multi-Scale (trainable)
FPN-style top-down pathway with **attention gates** at each merge:

```
f16 ─1×1─▶ p16 ───────────────┐
                               ├─ upsample ×2 ─▶ ⊕ ─ AG ─▶ p8
f8  ─1×1─▶ ──────────────────┘
                               ├─ upsample ×2 ─▶ ⊕ ─ AG ─▶ p4
f4  ─1×1─▶ ──────────────────┘
p4 ─ 3×3 conv ×2 ─▶ d [B,64,128,128] ─ upsample ×4 ─▶ [B,64,512,512]
```

- **Attention Gate (AG):** a lightweight spatial gate `σ(conv(relu(conv(g)+conv(x))))` that lets the higher-level (semantic) feature *suppress* irrelevant low-level activations — this is what helps the model ignore cars/shadows on the road surface while keeping the road. Cheap (a few convs), high payoff for occlusion.
- Each stage: `Conv3×3 → GroupNorm → GELU`. GroupNorm (not BatchNorm) because tile batches are small.
- Channel widths kept narrow (256→128→64) to stay within 16 GB.

### 3.4 Prediction heads (trainable)
Two sibling heads on the shared decoder feature `d`:

| Head | Layers | Output | Target |
|------|--------|--------|--------|
| **Road** | `Conv3×3 → GN → GELU → Conv1×1` | `[B,1,512,512]` logits | road binary mask |
| **Junction** | same shape | `[B,1,512,512]` logits | junction heatmap (Gaussian blobs at intersections) |

Two heads (not one multi-class) because road and junction are **non-exclusive** — a junction pixel is *also* a road pixel. Separate sigmoids model them independently.

---

## 4. Junction labels (auto-generated, no manual annotation)

We never hand-label junctions. They are derived from the road mask:

```
road_mask ─ skeletonize ─▶ 1-px skeleton
          ─ neighbor-count convolution (3×3) ─▶ count crossings
          ─ pixels with ≥3 skeleton neighbors = junction seeds
          ─ place 2-D Gaussian (σ≈3px) at each seed ─▶ junction_heatmap target
```

This makes the junction head a **free supervision signal** that directly serves Stage 2's junction-guided skeleton refinement and Stage 3's healing seeds. See [math.md](math.md#skeletonization) for the skeleton math.

---

## 5. Loss function

```
L = L_road + λ_j · L_junction

L_road  =  α·Dice  +  β·Lovász-softmax  +  γ·Boundary
L_junction =  MSE(junction_heat, target)   (+ optional focal weighting)

defaults: α=1.0, β=1.0, γ=0.5, λ_j=0.5
```

| Term | Purpose | Intuition |
|------|---------|-----------|
| **Dice** | Overlap on thin, class-imbalanced roads | Optimizes IoU directly; robust to the fact that roads are a small % of pixels |
| **Lovász-softmax** | Smooth surrogate for IoU | Fixes Dice's gradient behavior at the boundary of connectedness; better thin-structure recovery |
| **Boundary loss** | Crisp, continuous edges | Penalizes errors weighted by distance-to-boundary → discourages gaps that break topology |
| **Junction MSE** | Heatmap regression | Standard keypoint-style heatmap loss; Gaussian targets give graceful localization |

Full equations and gradients in [math.md](math.md). The **boundary + Lovász** pairing is deliberately topology-friendly: both punish the *breaks* that would later fragment the graph, reducing the load on Stage 3 healing.

---

## 6. Training pipeline (single mid GPU)

```
GeoTIFF + road vector labels (rasterized)
        │
   tile (512², overlap 64)  ──▶  albumentations (flip, rot90, scale, brightness,
        │                         CLAHE, simulated-shadow/occlusion cutout)
        ▼
   DataLoader (num_workers, pin_memory)
        ▼
   forward: encoder(no_grad) → decoder → heads
        ▼
   loss (Dice+Lovász+Boundary+Junction), AMP autocast
        ▼
   backward (only decoder+heads) → AdamW (lr 3e-4, cosine, warmup)
        ▼
   val every N steps: IoU, F1, APLS-lite, junction PR
        ▼
   checkpoint best (by APLS-lite) → artifacts/models/toposam_rr_{tag}.pt
```

- **AMP** (fp16) halves memory → larger tiles/batch on 16 GB.
- **Encoder under `no_grad`** → its activations don't need storing for backprop, saving major memory.
- **Occlusion augmentation** (random shadow polygons, coarse dropout simulating canopy) is the single most important augmentation — it *trains the robustness we're claiming*.
- **Effective batch** via gradient accumulation if a physical batch of 8 doesn't fit.

Details, schedules, and the dataset/fallback ladder are in [training.md](training.md).

## 7. Inference pipeline

```
Full raster
   │ windowed tiling (512², overlap 64)
   ▼
batched forward (AMP, no_grad)  ──▶ per-tile road_prob, junction_heat
   │ cosine seam-blend on overlaps (stitch.py)
   ▼
full-scene road_prob.tif, junction.tif  (georeferenced, written via rasterio)
   │ (optional) TTA: average flips/rot90 for final runs
   ▼
artifact registry  ──▶ consumed by Graph plane (Stage 2)
```

- **Seam blending:** overlapping tiles are feathered with a cosine window so road continuity isn't broken at tile borders — critical because a seam gap becomes a graph break.
- **Georeferencing preserved:** output `.tif` carries the input CRS/transform so the graph lands in real-world coordinates.

## 8. Feature-map size table

| Stage | Tensor | Shape (B=batch) |
|-------|--------|-----------------|
| Input | x | `[B, 3, 512, 512]` |
| Encoder neck | f16 | `[B, 256, 32, 32]` |
| ViT tap → proj | f8 | `[B, 128, 64, 64]` |
| ViT tap → proj | f4 | `[B, 64, 128, 128]` |
| Decoder out | d | `[B, 64, 128, 128]` → up `[B,64,512,512]` |
| Road head | road_logits | `[B, 1, 512, 512]` |
| Junction head | junction_logits | `[B, 1, 512, 512]` |

## 9. Fallback ladder (de-risking the model)

The model plane degrades gracefully — there is *always* a working predictor:

1. **Best:** TopoSAM-RR (frozen SAM + decoder + both heads), fine-tuned.
2. **If SAM too heavy / slow:** swap encoder for `timm` ConvNeXt-T or SMP EfficientNet-b3 (same decoder/heads).
3. **If training stalls:** ship a U-Net (`segmentation-models-pytorch`) trained on the same tiles — lower accuracy, identical downstream contract.
4. **If GPU dies at demo:** serve the **pre-baked inference cache** (`road_prob.tif` already in object storage). The graph/sim/dashboard run unchanged.

Because every fallback emits the *same two artifacts* (`road_prob.tif`, `junction.tif`), nothing downstream changes. This contract is the reason the project is safe to demo.

## 10. Code map

```
ml/toposam_rr/
├── models/
│   ├── encoder_sam.py      # frozen SAM wrapper + intermediate taps
│   ├── decoder_fpn.py      # hierarchical decoder + attention gates
│   ├── heads.py            # road & junction heads
│   └── toposam_rr.py       # assembles encoder+decoder+heads
├── losses/
│   ├── dice.py  lovasz.py  boundary.py  junction.py  combined.py
├── data/
│   ├── tiler.py  transforms.py  dataset.py  junction_labels.py
├── engine/
│   ├── trainer.py  evaluator.py  schedulers.py
└── inference/
    ├── predictor.py  stitch.py  export_onnx.py
```
