# YoTeLoLlevo — High-Performance Logistics Backend

A logistics system capable of validating delivery addresses against 1.5 million 
BOE records in under 1ms, optimizing multi-stop routes in under 15ms, and 
managing physical load order using a LIFO stack strategy.

Built as a deep-dive into data structures, concurrency, and algorithmic 
trade-offs in Java.

---

## The Problem

Three independently hard problems that turned out to be deeply connected:

1. **Address validation at scale** — check user-typed natural language addresses 
   against 1,547,893 fixed-width BOE records in under 100ms.
2. **Route optimization** — given N delivery stops, find the near-optimal 
   sequence minimizing total distance in under 15ms.
3. **Physical load management** — ensure the first package to be delivered 
   is the last one loaded (LIFO), eliminating reordering at each stop.

The interesting constraint: the data structure chosen for address indexing 
directly affected geocoding speed, which directly affected route optimization 
quality. These three problems could not be solved in isolation.

---

## Architecture & Key Technical Decisions

### 1. Address Indexing — Why a 3-Level Hierarchical Index

The naive approach was a flat HashMap with composite keys (`"28_115_02492"`).
Benchmarking revealed the problem immediately:

| Metric         | Flat HashMap | Hierarchical Index | Improvement |
|----------------|-------------|-------------------|-------------|
| Load time      | 8.3s        | 4.2s              | 49% faster  |
| Memory         | 180MB       | 120MB             | 33% less    |
| Search latency | 15ms (p50)  | 0.8ms (p50)       | 18x faster  |
| Throughput     | 18 req/s    | 35 req/s          | 94% more    |

The issue with flat keys: constructing 1.5M strings via `String.format()` 
forces Java to allocate memory, copy bytes, and compute hashes for each one. 
The 0.75 load factor causes ~600,000 hash collisions, degrading lookups to O(k).

Spanish addresses follow a natural 3-level hierarchy: 52 provinces → ~154 
municipalities each → hundreds of streets per municipality. Modeling this 
directly with nested `ConcurrentHashMap` structures gave three advantages:

- **Natural partitioning**: each province is an independent container, 
  enabling parallel writes with zero contention between threads.
- **Cache locality**: repeated lookups within the same municipality hit 
  CPU cache, not main memory.
- **Lock-free writes**: independent maps per province mean threads never 
  block each other.

### 2. Parallel Loading — Targeting P-Cores Explicitly

The hierarchical index alone was not sufficient. The real breakthrough was 
splitting the 1.5M-line file into 52 independent batches (one per province) 
and processing them in parallel using a thread pool pinned to P-Cores only, 
avoiding efficiency core saturation.

For record counting, `AtomicLong` was replaced with `LongAdder`. The 
difference: `AtomicLong` forces CPUs to communicate on every increment 
(cache line contention), while `LongAdder` maintains per-thread local 
counters and sums at the end only.

Final load result:
```
LOAD_DONE | MODE: OPT | SUCCESS: 1.519.034 | ERRORS: 0 
| LATENCY: 2885.78ms | THROUGHPUT: 526.386 reg/s | STATUS: READY
```

### 3. Normalization Cache — From Lock Contention to Concurrency

Address normalization (remove accents, uppercase, strip prepositions) costs 
~500 microseconds per string. With 3M operations at load time, that is 25 
minutes unoptimized.

First attempt: LRU cache using `LinkedHashMap` wrapped in 
`Collections.synchronizedMap`. This introduced a global lock on every 
read and write, forcing 11 threads to queue — eliminating all parallelism gains.

Final solution: `ConcurrentHashMap` as normalization cache. Trade-off 
accepted: losing strict LRU eviction order. Trade-off justified: hit rate 
remained at 85%, and throughput went from 434K to 787K reg/s.

| Implementation        | Throughput   | Hit Rate | Total Time  |
|----------------------|-------------|----------|-------------|
| No cache             | 0.66K reg/s | 0%       | ~25 min     |
| LinkedHashMap + sync | 434K reg/s  | 78%      | 385s        |
| ConcurrentHashMap    | 787K reg/s  | 85%      | 210s        |

### 4. Route Optimization — Hybrid NN + 2-Opt

The Travelling Salesman Problem for 20 stops has 2.43 × 10¹⁸ possible 
routes. Brute force is not an option. The goal: a near-optimal solution 
in under 15ms.

**Data structure decision first**: adjacency matrix over adjacency list.
In an optimization algorithm like 2-Opt, distances are queried millions 
of times. Matrix gives O(1) lookup vs O(n) for lists — up to 50x faster 
in practice due to direct indexing and CPU cache predictability.
Distances pre-computed once using the Haversine formula (~5ms), then 
reused throughout optimization.

**Two-phase algorithm:**

*Phase 1 — Nearest Neighbor (greedy construction)*: Start at warehouse, 
always move to the closest unvisited stop. O(n²), ~8ms for 50 packages. 
Produces routes 15-25% longer than optimal due to greedy myopia.

*Phase 2 — 2-Opt refinement*: Iteratively check if reversing any route 
segment reduces total distance. If crossing paths are detected, swap 
them permanently. Recovers 10-15% of distance lost in Phase 1.

| Algorithm          | Total Distance | vs Original | Execution Time |
|-------------------|---------------|-------------|----------------|
| Original route    | 145.2 km      | —           | —              |
| Nearest Neighbor  | 98.4 km       | -32%        | 8ms            |
| Hybrid (NN+2-Opt) | 87.1 km       | -40%        | 15ms           |
| Branch & Bound    | 85.8 km       | -41%        | 2300ms         |

The hybrid solution is 1.5% from mathematical optimum and 153x faster 
than exact search. In a production environment, that trade-off is obvious.

For real-world precision, Haversine distances are replaced with road 
distances from Mapbox Matrix API — a single API call returning the full 
NxN matrix for all active stops simultaneously. This improves route 
accuracy by 8-12%.

### 5. LIFO Stack for Physical Load Order

A mathematically optimized route (Warehouse → A → B → C) fails in 
practice if packages are loaded in visit order. With a single rear-access 
van, package A ends up at the bottom — requiring full unload and reload 
at every stop.

Solution: reverse the route before loading. Push C first, then B, then A. 
The delivery sequence becomes natural: pop A, deliver, pop B, deliver. 
Implemented via `Collections.reverse()` before the stack push phase.

---

## Final Performance Metrics

| Requirement         | Target    | Result        | Improvement |
|--------------------|-----------|--------------|-------------|
| Validation latency | < 100ms   | 0.8ms (p50)  | 99% better  |
| Throughput         | > 20 req/s| 35 req/s     | 75% better  |
| DB load time       | < 10s     | 4.2s         | 58% better  |
| Route improvement  | > 10%     | 18% average  | 80% better  |

---

## Stack

- Java 21
- Spring Boot (Maven)
- ConcurrentHashMap, LongAdder (java.util.concurrent)
- Mapbox Matrix API
- Haversine formula for in-memory distance pre-computation

---

## Key Takeaways

Architecture decisions outweigh algorithmic complexity. The choice of 
`ConcurrentHashMap` over `LinkedHashMap + synchronized` reduced processing 
time from minutes to seconds without changing a single algorithm. The best 
way to manage a shared resource in a concurrent system is often to not 
share it at all.
