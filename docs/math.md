# Mathematical Documentation

Every important algorithm in Route Resilience, explained **mathematically** and **intuitively**. Notation: images are functions on a pixel grid; graphs are `G=(V,E)` with edge weights `w(e)` (here `travel_cost`).

---

## Segmentation losses

### Dice loss
For predicted probabilities `p ∈ [0,1]` and binary target `g`:
```
Dice = (2 Σ pᵢgᵢ + ε) / (Σ pᵢ + Σ gᵢ + ε)
L_dice = 1 − Dice
```
**Intuition:** Dice ≈ soft IoU. It measures *overlap* relative to *total area*, so it is insensitive to the huge background. Roads are ~2–5% of pixels; a pixel-wise BCE would be dominated by background and happily predict "no road." Dice forces the model to actually overlap the thin road region. `ε` (≈1) avoids divide-by-zero on empty tiles.

### Lovász-softmax loss
A convex, differentiable surrogate for **1 − IoU** built from the **Lovász extension** of the IoU set function. For the foreground class, sort pixels by descending error `mᵢ = |gᵢ − pᵢ|`, then:
```
L_lovász = Σᵢ mᵢ · g_Δ(πᵢ)
```
where `π` is the error-sorted order and `g_Δ` is the discrete derivative of the **Jaccard loss** along that order (the "Lovász gradient").
**Intuition:** Dice's gradient gets weak exactly where boundaries are thin/uncertain. Lovász directly optimizes the IoU *ranking* of pixel errors, giving stronger, better-shaped gradients on thin structures — fewer broken roads. It complements Dice rather than replacing it.

### Boundary loss
Let `φ_G` be the **signed distance transform** of the ground-truth boundary (negative inside the road, positive outside). Then:
```
L_boundary = Σᵢ φ_G(i) · pᵢ
```
**Intuition:** errors are weighted by *how far they are from the true boundary*. A false prediction deep in the background costs more than one hugging the edge; a missed pixel right where a road should continue costs heavily. This penalizes the *gaps and blobs* that fragment topology, pushing the model toward continuous, crisp roads. Used with a small weight (γ=0.5) so it sharpens rather than destabilizes.

### Junction heatmap loss
Target `H*` is a sum of 2-D Gaussians (σ≈3) centered at junction pixels; prediction `H`:
```
L_junction = (1/N) Σᵢ (Hᵢ − H*ᵢ)²        # MSE
```
Optional focal weighting up-weights the sparse positive (junction) pixels.
**Intuition:** standard keypoint heatmap regression. Gaussian targets make localization smooth and tolerant of ±1–2 px label noise, which suits auto-generated junction labels.

### Combined objective
```
L = α·L_dice + β·L_lovász + γ·L_boundary + λ_j·L_junction
defaults α=β=1, γ=0.5, λ_j=0.5
```
Dice = overlap, Lovász = IoU-shaped gradients, Boundary = topology-friendly sharpness, Junction = structural seeds.

---

## Image → topology

### Hysteresis thresholding
Two thresholds `t_low < t_high`. Define strong `S = {p ≥ t_high}` and weak `W = {p ≥ t_low}`. Keep a weak pixel iff it is **8-connected to a strong pixel** (morphological reconstruction of `S` under mask `W`):
```
road = Reconstruct(S | W)
```
**Intuition:** a single threshold can't separate "faint real road under canopy" from "faint noise." Hysteresis keeps faint pixels *only when they continue from confident road*, which is exactly occlusion recovery. (Same principle as Canny edge linking.)

### Skeletonization (Zhang–Suen)
Iteratively delete boundary pixels of the binary road region while preserving connectivity, until a 1-px medial line remains. A foreground pixel `p` with 8-neighbors is deleted in a sub-iteration if:
```
(a) 2 ≤ B(p) ≤ 6        # B(p)=# foreground neighbors
(b) A(p) = 1            # A(p)=# 0→1 transitions in the ordered neighborhood
(c),(d) directional corner conditions (alternate each sub-iteration)
```
**Intuition:** peel the shape symmetrically from both sides so the surviving line stays centered and connected. `B(p)≥2` keeps it from breaking; `A(p)=1` forbids deleting a connecting pixel. The **medial axis** alternative additionally yields the distance-to-edge → road width.

### Junction detection on a skeleton
Convolve the skeleton with a 3×3 ones-kernel; a skeleton pixel with neighbor-count `≥3` is a junction (degree ≥3), `=1` is an endpoint. Snap to the local maximum of the junction heatmap within radius `r` for sub-skeleton accuracy.

---

## Graph construction & healing

### Connection Score (healing)
For a candidate gap-edge between endpoints `a,b`:
```
S_distance = exp(−‖a−b‖ / τ_d)
S_angle    = max(0, cos Δθ)^p              # Δθ = angle between a's road heading and the a→b direction (and symmetrically for b)
S_segconf  = (1/L) ∫₀ᴸ road_prob(a + t·(b−a)/L) dt     # mean prob along the gap
S_junction = max( junction_conf(near a), junction_conf(near b) )

score(a,b) = S_distanceᵂᵈ · S_angleᵂᵃ · S_segconfᵂˢ · S_junctionᵂʲ
cost(a,b)  = −log(score(a,b) + ε)
```
**Intuition:** multiply independent "is this a real road link?" evidences (a soft logical AND). Taking `−log` converts the multiplicative score into an **additive edge weight** so Kruskal/MST can optimize it. A link is cheap only when it is short **and** angularly continuous **and** supported by faint road evidence **and** (ideally) anchored at a junction.

### Union-Find (disjoint set)
Maintains a partition of nodes into components with two near-`O(1)` ops:
```
find(x):  follow parent pointers to the root, compressing the path
union(x,y): attach the shorter tree under the taller (union by rank)
```
With path compression + union by rank, `m` operations cost `O(m·α(n))`, where `α` is the inverse-Ackermann function (≤4 in practice).
**Intuition:** a fast "are these two endpoints already in the same road component?" test — the engine of Kruskal's MST.

### Minimum Spanning Tree (Kruskal)
Given candidate gap-edges with weights `cost(e)`:
```
sort edges by cost ascending
for e=(a,b) in order:
    if find(a) ≠ find(b):   # joining two different components
        accept e; union(a,b)
stop when one component remains
```
Produces a spanning forest→tree of minimum total weight.
**Intuition:** add the **cheapest** healed links that *increase connectivity*, skipping any that would just form a cycle. Result: the minimum set of most-plausible reconnections — connectivity with no redundant or contradictory healed roads. Correctness: the **cut property** — the lightest edge crossing any cut is safe to include.

---

## Network intelligence

### Shortest paths (Dijkstra)
With non-negative weights `w(e)=travel_cost`, Dijkstra computes single-source shortest paths in `O(E + V log V)` (binary/Fibonacci heap). Foundation for betweenness, efficiency, and rerouting.
**Intuition:** greedily finalize the closest-unvisited node; because weights are non-negative, its distance can't later improve.

### Betweenness centrality
For node `v`:
```
C_B(v) = Σ_{s≠v≠t}  σ_st(v) / σ_st
```
`σ_st` = number of shortest s–t paths; `σ_st(v)` = those passing through `v`. **Edge betweenness** is the analogous sum over edges. Computed with **Brandes' algorithm** in `O(VE)` (unweighted) / `O(VE + V² log V)` (weighted); **approximated** by sampling `k` source nodes → `O(kE …)`.
**Intuition:** the share of all optimal trips that must route through `v` (or `e`). High betweenness = a bottleneck the network depends on. This is the primary criticality signal.

### Articulation points & bridges (DFS low-link / Tarjan)
DFS assigns discovery times `disc[u]` and `low[u] = min(disc[u], disc over back-edges, low over children)`.
```
edge (u,child) is a BRIDGE          iff  low[child] > disc[u]
non-root u is an ARTICULATION POINT  iff  ∃ child with low[child] ≥ disc[u]
root is articulation                 iff  it has ≥2 DFS children
```
`O(V+E)` single pass.
**Intuition:** `low[child] > disc[u]` means the subtree under `child` has **no alternative route** back above `u` — cut that edge/node and the graph splits. These are the binary, no-redundancy vulnerabilities a planner must protect first.

### Community detection (Louvain)
Maximize **modularity**:
```
Q = (1/2m) Σ_{ij} [ A_ij − (k_i k_j)/(2m) ] · δ(c_i, c_j)
```
`A` = adjacency (weighted), `k_i` = degree of `i`, `m` = total weight, `c_i` = community of `i`. Louvain greedily moves nodes to neighboring communities to increase `Q`, then aggregates communities into super-nodes and repeats.
**Intuition:** `Q` rewards more intra-community edges than you'd expect by chance. The result is the city's natural **districts**; the few edges *between* districts are arteries → high criticality.

### Urban Criticality Score
```
crit(e) = w_b·n(C_B(e)) + w_r·𝟙[bridge] + w_a·𝟙[touches articulation] + w_d·n(load)
n(·) = min-max normalization to [0,1];   Σ w = 1
```
**Intuition:** a single interpretable `[0,1]` blend where categorical "no alternative" failures (bridge, articulation) outrank gradual load. Tunable so a planner can re-weight to their priorities.

---

## Resilience & simulation

### Network efficiency (global)
```
E(G) = (1 / (N(N−1))) Σ_{i≠j} 1 / d(i,j)
```
`d(i,j)` = shortest-path distance (∞ if disconnected → contributes 0).
**Intuition:** average "ease of travel." Robust to disconnection (a broken pair just adds 0), unlike average path length which becomes ∞. Our primary scalar for "how good is the network right now."

### Resilience Index
Compare efficiency before vs after a perturbation (failure scenario):
```
R = E(G_after) / E(G_before)        ∈ [0,1]
```
Or as a curve: remove the top-`k` critical elements and plot `E` vs `k` — the **area under the resilience curve** summarizes robustness.
**Intuition:** `R=1` means the failure didn't hurt (redundancy exists); `R→0` means the failure crippled the network. Removing high-betweenness/bridge edges and watching `R` fall *validates* the criticality scoring — the very roads we flagged are the ones whose loss matters.

### Average (reachable) path length & travel delay
```
L_avg = mean over connected pairs of d(i,j)
delay = L_avg(after) − L_avg(before)        # extra travel cost imposed by the failure
```
Computed on the **largest connected component** to stay finite under partial disconnection.

### Connected components / fragmentation
Number and sizes of connected components after a scenario; the **largest-component fraction** `|LCC|/N` measures how badly the network shattered.
**Intuition:** a flood that splits the city into islands shows up as a sudden drop in `|LCC|/N` — an immediately legible "the city is cut in two" signal for the dashboard.

---

## Why these choices

- **Losses** chosen to optimize *topology*, not just pixels (Dice+Lovász+Boundary all attack fragmentation).
- **Hysteresis + skeleton + junction prior** recover faint roads and place intersections accurately.
- **Connection Score + MST** heal with model belief, minimally and plausibly.
- **Betweenness + bridges + articulation + Louvain** are the standard, well-understood vocabulary of network criticality — credible to reviewers, computable in our time budget.
- **Efficiency-based resilience** stays finite under disconnection, which is precisely the regime stress tests create.
