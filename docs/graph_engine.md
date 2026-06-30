# Graph Engine — Stage 3 (Healing), Stage 4 (Attributes), Stage 5 (Network Intelligence)

The graph engine turns the **raw, disconnected** road graph from Stage 2 into a **connected, weighted, analyzed** network with a per-edge/per-node **criticality score**. This is the heart of the project's value: the planner doesn't want a mask, they want to know *which roads matter and which are fragile*.

---

## Stage 3 — Confidence-Guided Graph Healing

Occlusions leave the raw graph fragmented into components. Naively connecting nearest endpoints (plain MST on Euclidean distance) produces *physically wrong* roads (cutting across buildings, ignoring road direction). We instead score each candidate reconnection with a **Connection Score** that fuses geometry *and* model confidence, then build an MST over those scores.

```
graph_raw (disconnected)
   │
   ▼ ┌──────────────────────────────────────────────┐
     │ 1. Candidate Edge Generator   (KD-tree)        │  endpoint pairs within max_gap
     │ 2. Connection Scoring Engine                   │  distance·angle·seg·junction
     │ 3. Graph Optimizer (Union-Find + MST)          │  connect components cheaply
     │ 4. Edge Realizer                               │  add healed edges, flag them
     └──────────────────────────────────────────────┘
   ▼
graph_healed (connected)   → Stage 4
```

### 1. Candidate Edge Generator — `graph/healing/candidates.py`
- Collect all **dangling endpoints** (degree-1 nodes) — these are where roads were cut.
- Build a **KD-tree** over endpoint coordinates; for each endpoint, query neighbors within `max_gap_px`.
- Emit candidate pairs `(a, b)` that belong to **different components** (a within-component link adds a cycle, not connectivity — skip unless it's a plausible loop closure).

Complexity: KD-tree neighbor query is `O(n log n)` over endpoints — endpoints are a tiny fraction of nodes, so this is cheap.

### 2. Connection Scoring Engine — `graph/healing/scoring.py`

Each candidate gets a **cost** (lower = better connection). The Connection *Score* is high for good links; we minimize its negative (a cost) in the MST.

```
score(a,b) =  w_d · S_distance
            · w_a · S_angle
            · w_s · S_segconf
            · w_j · S_junction

S_distance  = exp(−dist(a,b) / τ_d)                 # closer is better
S_angle     = max(0, cos(Δθ))^p                     # continuing the road's heading is better
S_segconf   = mean road_prob sampled along segment a→b   # does the model 'see' road in the gap?
S_junction  = max junction_conf near a or b          # endpoints at a predicted junction connect more readily

cost(a,b)   = −log(score(a,b) + ε)                   # turn product-score into additive MST weight
```

| Factor | Intuition | Why it matters for occlusion |
|--------|-----------|------------------------------|
| **Distance** | Real road gaps are short | Avoids absurd long-range links |
| **Angular continuity** | Roads don't kink 90° across a gap | Connects the *correct* opposing endpoint, not just the nearest |
| **Segmentation confidence in the gap** | If the model has *faint* road in the occluded gap, it's probably a real road | Directly leverages the occlusion-robust prob map — faint-but-present evidence |
| **Junction confidence** | Endpoints near a predicted intersection should merge there | Uses Stage 1's junction head as a structural prior |

This is the project's signature algorithm: **healing is driven by what the model believes, not just by geometry.** Full math: [math.md#connection-score](math.md).

### 3. Graph Optimizer — Union-Find + MST — `graph/healing/optimizer.py`
- Treat each existing connected component as a super-node via **Union-Find (disjoint set, path compression + union by rank)**.
- Sort candidate edges by `cost` ascending; **Kruskal's MST**: add an edge only if it joins two different components (Union-Find `find` differ) → union them.
- Stop when all components are merged (single component) or no candidate remains under threshold.

**Why MST, not "connect everything cheap enough"?** MST adds the **minimum number of healed edges** to achieve connectivity, each chosen as the *cheapest* (most plausible) link between two components — no redundant or contradictory healed roads. Plain proximity-joining would create a hairball. Math + correctness intuition: [math.md#mst](math.md), [math.md#union-find](math.md).

### 4. Edge Realizer — `graph/healing/optimizer.py`
- Materialize each accepted candidate as a real edge with a straight (or angle-interpolated) polyline.
- **Flag it `healed: true`** with its score → the dashboard renders healed segments distinctly (honesty + explainability) and analytics can down-weight uncertain links.

**Fallback:** if scoring degenerates (e.g., junction head missing, all scores ~equal), fall back to plain Euclidean MST (`fallback: mst` in config). Connectivity is still achieved; only the *quality* of link selection drops.

**Output — `graph_healed`:** connected graph, healed edges flagged with scores.

---

## Stage 4 — Road Attribute Assignment

Turn the connected graph into a **weighted, routable** graph. `graph/attributes/assign.py`.

```
graph_healed
   │  per edge:
   ├── length_m      = polyline length × pixel_size_m                (geodesic-aware)
   ├── direction     = bearing of the edge (deg)                     atan2 of endpoints
   ├── curvature     = 1 − (straight-line dist / path length) ∈[0,1] sinuosity
   ├── confidence    = seg_conf (mean road_prob), low for healed     reliability
   ├── width_est     = mean medial-axis distance × 2 × pixel_size_m  (if available)
   └── travel_cost   = length_m / speed(width, curvature, healed)    routing weight
   ▼
graph_attributed (weighted)   → Stage 5
```

**Travel cost model** (transparent, tunable):
```
speed = base_speed_kmh
        × width_factor(width_est)        # wider ≈ faster (proxy for road class)
        × (1 − 0.3·curvature)            # curvy roads slower
        × (0.7 if healed else 1.0)       # uncertain links penalized
travel_cost = length_m / speed
```
This makes shortest-path routing prefer confident, straight, arterial-like roads — sensible defaults a planner can override.

**Output — `graph_attributed`:** NetworkX graph; nodes carry geo coords, edges carry the attribute dict above. Persisted as `graph.gpkg` (GIS) + `graph.json` (frontend).

---

## Stage 5 — Network Intelligence

Compute the analytics that answer *"which roads are critical?"* `graph/analytics/`.

```
graph_attributed
   │
   ├── centrality.py     Betweenness (node) + Edge Betweenness
   ├── connectivity.py   Articulation points + Bridges
   ├── community.py      Louvain community detection
   └── criticality.py    weighted fusion → Urban Criticality Score
   ▼
criticality layers (per node & per edge)   → API → dashboard
```

### 5.1 Betweenness centrality — `centrality.py`
Fraction of shortest paths (weighted by `travel_cost`) passing through a node/edge. **High betweenness = a road many trips depend on = a bottleneck.**
- Exact for small graphs; **approximate via k sampled sources** (`approx_k`) for large graphs — near-linear scaling.
- Optional **igraph** backend for the heavy call (≈100× faster than NetworkX on big graphs).
- Math + intuition: [math.md#betweenness](math.md).

### 5.2 Articulation points & bridges — `connectivity.py`
- **Articulation point:** a node whose removal disconnects the graph → a single-point-of-failure intersection.
- **Bridge:** an edge whose removal disconnects the graph → a road with *no alternative*.
- Computed in `O(V+E)` via DFS low-link (NetworkX). These are the **hard** vulnerabilities — losing one splits the city. Math: [math.md#articulation-points](math.md).

### 5.3 Community detection — `community.py`
- **Louvain** modularity maximization → partitions the city into neighborhoods/zones.
- **Inter-community edges** are the arteries linking districts → strong criticality candidates.
- Math: [math.md#community-detection](math.md).

### 5.4 Urban Criticality Score — `criticality.py`
Fuse the signals into one interpretable per-edge (and per-node) score in `[0,1]`:

```
criticality =  w_b · norm(betweenness)
             + w_r · is_bridge
             + w_a · is_articulation          (node) / touches_articulation (edge)
             + w_d · norm(degree / load)

defaults: w_b=0.4, w_r=0.3, w_a=0.2, w_d=0.1   (configurable)
```
- Each term normalized to `[0,1]`; weights sum to 1 → score is interpretable.
- **Bridges and articulation points dominate** by design — they're categorical "no redundancy" failures that matter more than gradual load.

**Output — criticality layers:** `centrality.json`, `bridges.json`, `articulation.json`, `communities.json`, `criticality_score.json`. The dashboard renders these as the **criticality heatmap**; the simulation plane uses them to prioritize stress-test targets.

---

## Performance & scaling notes

| Operation | Cost | Strategy |
|-----------|------|----------|
| Healing candidate gen | `O(n log n)` (KD-tree, endpoints only) | cheap |
| MST | `O(E log E)` (Kruskal) | cheap |
| Betweenness (exact) | `O(V·E)` | sample-k approx + igraph for big graphs |
| Articulation/bridges | `O(V+E)` | always exact |
| Louvain | near-linear | fine |
| **Caching** | — | all Stage-5 outputs cached per `(scene, params_hash)`; recomputed only on param change |

The whole Stage 3–5 chain is **deterministic and CPU-only**, so it runs without a GPU and is fully cacheable — the dashboard reads results instantly. Interactive simulation ([simulation.md](simulation.md)) operates on this cached graph and recomputes only the *affected* metrics.

## Module map

```
graph/
├── healing/    candidates.py  scoring.py  optimizer.py
├── attributes/ assign.py
└── analytics/  centrality.py  connectivity.py  community.py  criticality.py
```
