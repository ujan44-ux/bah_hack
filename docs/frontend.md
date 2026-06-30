# Frontend — Stage 7: Planner Dashboard

A **Next.js + TypeScript** single-page dashboard. The map uses **MapLibre GL** (base map, vector tiles) with **deck.gl** overlays (GPU-rendered roads, criticality heatmap, paths). State is **Zustand**; server state/caching is **TanStack Query**; charts are **Recharts**; UI is **Tailwind + shadcn/ui**.

> Why this stack: deck.gl renders 100k+ edges on the GPU (Leaflet's DOM rendering dies past a few thousand). MapLibre is the open, token-free base layer. The combo looks production-grade and stays smooth during live simulation. (See [tech_stack.md](tech_stack.md).)

---

## 1. Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│ TopBar:  scene selector · pipeline status (live) · layer toggles       │
├───────────────┬──────────────────────────────────────────┬───────────┤
│ LeftPanel     │                                          │ RightPanel │
│ ───────────   │              MAP CANVAS                   │ ───────────│
│ • Layers      │   MapLibre base + deck.gl overlays:       │ Metrics:   │
│   - road net  │     - RoadLayer (confidence-colored)      │ • resilience
│   - healed    │     - CriticalityLayer (heatmap)          │   index    │
│   - junctions │     - BridgeLayer / ArticulationLayer     │ • efficiency
│ • Scenario    │     - FloodPolygon (draw)                 │ • Δ path   │
│   builder     │     - RouteLayer (before/after paths)     │ • components
│ • Run / Reset │                                          │ • charts   │
└───────────────┴──────────────────────────────────────────┴───────────┘
                          BottomBar: resilience curve · timeline scrubber
```

## 2. Component hierarchy

```
<App>
└── <DashboardLayout>
    ├── <TopBar>
    │   ├── <SceneSelector/>         // pick/upload scene
    │   └── <JobStatus/>             // SSE-driven pipeline progress
    ├── <LeftPanel>
    │   ├── <LayerToggles/>          // show/hide overlays
    │   └── <ScenarioBuilder>
    │       ├── <ScenarioTypePicker/> // node/road/flood/congestion
    │       ├── <PolygonDrawTool/>    // draw flood region on map
    │       └── <RunControls/>        // run / reset / compare
    ├── <MapView>                     // MapLibre + deck.gl host
    │   ├── <RoadLayer/>
    │   ├── <CriticalityHeatmap/>
    │   ├── <BridgeArticulationLayer/>
    │   ├── <FloodPolygonLayer/>
    │   ├── <RouteLayer/>             // before/after reroute animation
    │   └── <Tooltip/>                // hover edge/node details
    ├── <RightPanel>
    │   ├── <MetricCards/>            // resilience index, efficiency, delay
    │   └── <MetricCharts/>           // Recharts: histograms, before/after
    └── <BottomBar>
        └── <ResilienceCurve/>        // efficiency vs k removed (targeted/random)
```

## 3. State management (Zustand)

```ts
interface AppState {
  sceneId: string | null;
  layers: { roads: boolean; healed: boolean; junctions: boolean;
            criticality: boolean; bridges: boolean; articulation: boolean };
  scenario: ScenarioSpec | null;        // type + params (e.g. drawn polygon)
  simResult: SimulationResult | null;   // latest stress-test output
  selected: { kind: 'node'|'edge'; id: number } | null;
  setScenario, setLayer, setSimResult, selectFeature, reset
}
```
- **Zustand** for *client* state (toggles, drawn polygon, selection) — minimal boilerplate.
- **TanStack Query** for *server* state (graph, analytics, sim results) — caching, dedupe, background refetch. Clean split: Zustand = UI intent, Query = server truth.

## 4. Map rendering

- **Base:** MapLibre GL with a vector/raster basemap; the segmentation probability map can be shown as a raster overlay (TileJSON from the API).
- **Overlays (deck.gl `MapboxOverlay` interleaved with MapLibre):**
  - `PathLayer` for roads, **colored by `confidence` or `criticality`** (toggle), width by road class.
  - `HeatmapLayer` / `PathLayer` colored by criticality for the heatmap.
  - `ScatterplotLayer` for junctions / articulation points; thick `PathLayer` for bridges.
  - `PolygonLayer` + `EditableGeoJsonLayer` (nebula.gl) for **drawing flood regions**.
  - Animated `TripsLayer`/`PathLayer` for **before/after reroutes**.
- **Picking:** deck.gl GPU picking → hover/click an edge or node → `<Tooltip>` with attributes (length, confidence, betweenness, healed?).

## 5. Simulation controls (the interactive core)

```
User picks scenario type → (draws polygon / clicks roads / sets multiplier)
   → scenario stored in Zustand
   → "Run" → POST /scenes/{id}/simulate
   → simResult → overlays recolor (affected edges red), metrics + charts update,
     reroute paths animate.  "Compare" splits map into before/after.
```
Because the API simulates on the cached graph, the round-trip is sub-second → the slider/redraw feels live.

## 6. Live updates

- **Job progress:** subscribe to `GET /jobs/{id}/stream` (SSE) → `<JobStatus>` shows per-stage progress (infer → graph → analytics) in real time.
- **No polling for sim:** simulation is request/response and fast; results update via TanStack Query mutation.

## 7. API integration layer

```
frontend/src/lib/api.ts        // typed fetchers (zod-validated responses)
frontend/src/hooks/
  useGraph(sceneId)            // GET /graph        (Query)
  useCriticality(sceneId)      // GET /analytics    (Query)
  useSimulate()                // POST /simulate    (Mutation)
  useJobStream(jobId)          // SSE subscription
```
Response shapes are validated with **zod** mirrors of the API schemas → geo payload bugs caught at the boundary.

## 8. Dashboard "wow" moments (demo script)

1. Load scene → watch the pipeline stages tick live (SSE).
2. Toggle **criticality heatmap** → the few red arterial roads pop.
3. Draw a **flood polygon** over a river district → metrics drop, city splits into components, affected roads glow red.
4. Pick an **ambulance O-D** → before/after routes animate, "+6.4 min" delay shown.
5. Run the **resilience curve** (targeted vs random) → targeted collapses fast, *proving* the criticality score found the right roads.

## 9. Module map

```
frontend/src/
├── pages/ (or app/)   dashboard route
├── components/        TopBar, MapView, panels, layers, charts
├── hooks/             useGraph, useCriticality, useSimulate, useJobStream
├── store/             zustand store
└── lib/               api.ts, deck-layers.ts, geo.ts, schemas.ts (zod)
```
