# Simulation — Stage 6: Stress Testing & Resilience

The simulation plane answers the planner's real question: **"What happens to the city if *this* fails?"** It applies a *scenario* (node failure, road closure, flooding, congestion) to the cached attributed graph, recomputes resilience metrics, and returns deltas — fast enough to feel interactive because it never touches the GPU or re-runs the upstream pipeline.

---

## 1. Design

```
graph_attributed (cached, in-memory)   +   Scenario spec (from UI)
        │
        ▼ ┌─────────────────────────────────────────────┐
          │ 1. Scenario Resolver   spec → affected V/E    │
          │ 2. Graph Perturbator   apply removal/reweight │  (copy-on-write)
          │ 3. Metric Engine       resilience deltas      │
          │ 4. Reroute Engine      new shortest paths      │
          └─────────────────────────────────────────────┘
        ▼
SimulationResult (JSON)   → API → dashboard overlays + charts
```

**Key performance principle:** the base graph and its baseline metrics are computed **once** and cached. A scenario operates on a **copy-on-write view** and recomputes only what the perturbation affects. Most scenarios return in well under a second on a city-scale graph → the dashboard slider feels live.

---

## 2. Scenario types — `simulation/scenarios/`

| Scenario | Spec | Effect on graph | Module |
|----------|------|-----------------|--------|
| **Node failure** | `node_ids[]` | Remove nodes + incident edges | `node_failure.py` |
| **Road closure** | `edge_ids[]` | Remove edges | `road_closure.py` |
| **Flooding region** | GeoJSON polygon + depth | Edges intersecting polygon → removed (or cost ×big if passable) | `flooding.py` |
| **Congestion** | region/edges + multiplier | `travel_cost ×= m` (no removal) | `congestion.py` |
| **Targeted attack** | `top_k` by criticality | Remove the k most-critical elements | derived |
| **Random failure** | `p`, `seed` | Remove each edge w.p. `p` | derived |

- **Flooding** uses `shapely`: build edge geometries, query a spatial index (STRtree) for polygon intersection → affected edges in `O(log n)` per edge. Passable-vs-impassable decided by a depth/clearance threshold.
- **Targeted vs random** is the classic robustness contrast: networks tolerate random loss but are fragile to *targeted* loss of high-criticality elements. Showing both **validates the criticality score** live.

### Scenario spec schema
```json
{
  "type": "flooding",
  "params": { "polygon": {"type":"Polygon","coordinates":[...]}, "depth_m": 1.2 },
  "compare_to": "baseline"
}
```

## 3. Graph Perturbator — `simulation/engine.py`
- **Copy-on-write:** `G_sim = G.copy()` (or a lightweight edge-filter view for large graphs to avoid full copies) → apply removals/reweights. The base graph is never mutated, so concurrent simulations from multiple users are isolated.
- Returns `G_sim` plus the set of directly affected nodes/edges (for incremental metrics + UI highlighting).

## 4. Metric Engine — `simulation/metrics/resilience.py`

Computes, before/after:

| Metric | Formula (see [math.md](math.md)) | Meaning |
|--------|-----------------------------------|---------|
| **Resilience Index** | `E(G_after)/E(G_before)` | Fraction of network efficiency retained |
| **Network efficiency** | `E(G)=1/(N(N−1)) Σ 1/d(i,j)` | Average ease of travel (finite under disconnection) |
| **Avg path length** | mean `d(i,j)` over LCC | Typical trip cost |
| **Travel delay** | `L_avg(after) − L_avg(before)` | Extra cost imposed |
| **Connected components** | count + `|LCC|/N` | How shattered the city is |
| **Lost demand** | fraction of node pairs now disconnected | Trips that become impossible |

**Incremental computation:** for single-element removals we recompute efficiency on the affected region where possible; for larger scenarios we recompute globally but on the cached graph (still CPU-cheap). Approximate betweenness (sampled-`k`) keeps re-scoring fast when the UI asks to refresh criticality post-failure.

## 5. Reroute Engine — `simulation/engine.py`
- Given an origin–destination pair (or a saved set of "key trips," e.g. hospital↔zones), compute shortest paths on `G_before` and `G_sim` (Dijkstra over `travel_cost`).
- Return both paths + the delay so the dashboard can **animate the detour** and quantify the cost ("ambulance route +6.4 min after this bridge floods").
- This is the single most persuasive demo moment — a concrete, human-legible consequence.

## 6. Output object — `SimulationResult`
```json
{
  "scenario": { "type": "flooding", "params": {...} },
  "metrics": {
    "resilience_index": 0.62,
    "efficiency_before": 0.041, "efficiency_after": 0.025,
    "avg_path_len_before": 1820, "avg_path_len_after": 2410, "travel_delay": 590,
    "components_before": 1, "components_after": 3, "lcc_fraction": 0.78,
    "lost_demand": 0.14
  },
  "affected": { "nodes": [...], "edges": [...] },
  "reroutes": [ {"od":[12,455], "before":[...], "after":[...], "delay": 380} ]
}
```
Cached per `(scene, scenario_hash)` so repeated/identical scenarios are instant.

## 7. Why this design wins

- **Interactive** — runs on the cached graph, no GPU, sub-second → a *live* planner tool, not a batch report.
- **Validating** — targeted vs random failure curves *prove* the criticality score points at the right roads.
- **Legible** — reroute animations + "travel delay in minutes" translate graph theory into language a planner/judge instantly gets.
- **Composable** — scenarios stack (flood + congestion) because each is just a graph transform.

## 8. Module map

```
simulation/
├── scenarios/   node_failure.py  road_closure.py  flooding.py  congestion.py
├── engine.py    perturbator + reroute
└── metrics/     resilience.py  (efficiency, delay, components, resilience curve)
```
