## Route Resilience  Full Backend 

### Technology choices & why they make it faster

| Decision        | Technology                          | Speed rationale                                                                 |
|-----------------|-----------------------------------  |---------------------------------------------------------------------------------|
| HTTP server     | Spring WebFlux (Reactor)            | Non-blocking I/O; one thread can serve hundreds of concurrent requests without blocking |
| Java version    | Java 21 (LTS)                       | Virtual threads (Thread.ofVirtual()) for cheap concurrency in ablation simulations |
| In-process cache| Caffeine                            | On-heap, near-zero-GC cache; graph reads return in ~2ms vs ~30ms cold file I/O   |
| Graph library   | JGraphT                             | Pure-Java graph algorithms (BFS, Dijkstra, betweenness) faster than calling Python |
| JSON            | Jackson with @JsonIncludeProperties | Skip serialising unused fields; 30–40% smaller payloads                          |
| File watching   | Java NIO WatchService               | Detects new Python outputs and hot-reloads cache automatically                   |
| Build           | Maven with -T4 parallel builds      | Cuts build time in half on quad-core                                             |
| API versioning  | /api/v1/ prefix                     | Future-proof; v2 endpoints can coexist without breaking clients                  |


Why it's fast the three key decisions:
1>The first is Spring WebFlux, rather than Spring MVC. WebFlux uses Netty under the hood with non-blocking I/O, so a single thread can handle hundreds of concurrent requests without idling while waiting for file reads or computations. All controllers return Mono<T> or Flux<T>  reactive types that chain work lazily.

2>The second is Caffeine in-memory cache. The road graph GeoJSON and centrality scores are pre-computed by Python and only read from disk once. After that, GET /graph returns in ~2ms because Caffeine serves directly from heap. A FileWatcherService using Java NIO's WatchService monitors the Python output directory and auto-evicts the cache the moment a new file appears  no restart needed.

3>The third is Java 21 virtual threads for ablation. Graph simulation (removing nodes, re running Dijkstra, computing path length ratios) is CPU bound and can take 200–800ms. Rather than blocking a Netty I/O thread (which would kill concurrency) or maintaining a traditional thread pool, the ablation executor uses Executors.newVirtualThreadPerTaskExecutor()  each ablation request gets its own virtual thread, scheduled by the JVM, with near zero overhead. A hundred simultaneous ablation requests don't queue up.

What every layer does:
The controller layer is intentionally thin  it only routes, validates, and delegates. The service layer holds all logic: GraphService and HeatmapService are pure cache wrappers, while AblationService contains the full JGraphT simulation pipeline (build graph, remove nodes, sample path lengths, find affected routes, count components). ScenarioController composes both services it asks HeatmapService for the top-N nodes, then passes them to AblationService as a normal ablation request, so scenarios get caching for free.
The ZGC GC flag is important here it keeps GC pauses under 1ms even as the graph objects are large, so the ~2ms cache hit latency isn't blown up by a GC pause mid response.


```mermaid
flowchart TD
    A[HTTP Request] --> B[WebFlux Router]
    B --> C[GraphController]
    C --> D[GraphService]
    D --> E{Caffeine Cache Hit?}

    E -- Yes --> F[~2ms → Mono<GeoJSON> file]
    E -- No --> G[~30ms Cold File Read → Populate Cache]

    %% Styling with better contrast
    style A fill:#f9f871,stroke:#333,stroke-width:2px,color:#000
    style B fill:#4dabf7,stroke:#1c1c1c,stroke-width:2px,color:#fff
    style C fill:#51cf66,stroke:#1c1c1c,stroke-width:2px,color:#fff
    style D fill:#ff922b,stroke:#1c1c1c,stroke-width:2px,color:#fff
    style E fill:#845ef7,stroke:#1c1c1c,stroke-width:2px,color:#fff
    style F fill:#20c997,stroke:#1c1c1c,stroke-width:2px,color:#fff
    style G fill:#fa5252,stroke:#1c1c1c,stroke-width:2px,color:#fff

