# Specialist: Performance & Scalability

Review scope: would this code degrade at scale — millions of requests, large datasets, high concurrency?

## Hot-path analysis

- Are expensive operations (DB queries, network calls, serialization, crypto) inside a loop or called on every request?
- Is computation repeated when a cached result would suffice?
- Are large intermediate collections created when streaming or lazy evaluation would work?

## N+1 patterns

- Does any new code fetch in a loop what could be retrieved in one batched query?
- Is an ORM relationship accessed in a loop without eager loading?
- Are sequential HTTP calls made where a batch endpoint exists?

## Memory allocation

- Unnecessary object creation in hot paths (per-request allocations that should be pooled)
- Large buffers allocated for every operation when they could be reused
- Collections that grow unbounded (caches without eviction, queues without backpressure)

## Concurrency

- Serial awaits in loops: `for item in items { await process(item) }` — should these be parallel?
- Connection pool exhaustion: new connections created per request instead of reusing a pool
- Lock contention: coarse-grained locks held across I/O operations
- Race conditions: check-then-act sequences without atomic guarantees

## Cardinality

- New metric, log field, or cache key with user-specific or unbounded values (high cardinality = memory explosion)
- Map/set sized proportional to unique users, sessions, or events without a bound

## Benchmark regressions

- If benchmarks exist and the diff touches their critical paths, are results expected to be neutral or better?

## Severity guidance

| Severity | Example |
|----------|---------|
| MUST | DB query or network call inside a loop on unbounded data |
| MUST | Unbounded memory growth (cache without eviction, queue without backpressure) |
| SHOULD | O(n²) algorithm on inputs that grow with user data |
| SHOULD | Repeated computation that could be cached with trivial effort |
| MAY | Minor allocation that is unlikely to matter at expected scale |
