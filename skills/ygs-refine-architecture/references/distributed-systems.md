# Distributed Systems Architecture Principles

## Domain-Driven Design
- **Bounded contexts** — Clear ownership boundaries. Each microservice owns one context.
- **Ubiquitous language** — Shared vocabulary between domain experts and developers. Same terms in code and conversation.
- **Aggregates** — Clustered entities functioning as a unit. Aggregate root controls lifecycle and consistency boundary.
- **Value objects** — Immutable, defined by attributes not identity. Share freely.
- **Domain events** — Capture state changes for async communication between bounded contexts.
- **Repository pattern** — Abstract persistence behind domain interfaces.
- **Context mapping** — Explicit integration patterns: Shared Kernel, Anticorruption Layer, Open Host Service.

## Architecture Patterns
- **Hexagonal (Ports & Adapters)** — Dependencies point inward. Domain knows nothing of infrastructure.
- **Clean Architecture** — Entities → Use Cases → Interface Adapters → Frameworks. Dependency rule: inward only.
- **CQRS** — Separate read and write models when patterns differ significantly.
- **Event Sourcing** — Events as source of truth. Complete audit trail, replay, temporal queries.
- **Functional core, imperative shell** — Pure domain logic, I/O at boundaries only.
- **Actor model** — Isolated mutable state, message-passing only. No locks, no races.

## Transaction Boundaries
- Atomicity requirements drive architecture — scope of atomic guarantees dictates boundaries
- Single-database ACID for most operations; SAGAs for distributed
- Outbox pattern eliminates dual-write (DB + event in same transaction, relay async)
- Change Data Capture (CDC) for sub-second event publishing without polling
- Event Sourcing when complete audit trail and replay are required

## Fault Tolerance Patterns
- **Circuit Breaker** — Stop requests to failing services, prevent cascade
- **Retry with Jitter** — Exponential backoff + randomization prevents thundering herd
- **Bulkhead Isolation** — Thread/connection pool separation per dependency
- **Graceful Degradation** — Partial functionality during dependency failure
- **Redundancy and Failover** — No single points of failure
- **Graceful Shutdown** — Drain in-flight requests on SIGTERM

## Consistency Models
- Evaluate strong vs. eventual consistency per operation
- Financial/ordering operations likely need strong consistency
- Read-heavy, latency-sensitive operations can use eventual
- MVCC understanding: long transactions hold snapshots, impact cleanup

## Caching Architecture
- Caching is not a silver bullet — adds complexity and staleness risk
- Pre-load caches before traffic spikes
- Implement both positive and negative caching
- Test performance without cache during peak (what if cache fails?)
- Cache keys MUST respect authorization boundaries
- TTL + explicit invalidation (not just one)
- Minimize bimodal logic (behavior different with/without cache)

## API Boundaries
- Clear internal/external API separation with different governance
- Separate control-plane from data-plane
- Idempotency tokens on all create operations
- Pagination with expiring tokens on all unbounded queries
- Versioning strategy before first client ships

## Operational Concerns
- Deep health checks (not just "200 OK" — check DB, cache, deps)
- Structured logging with correlation IDs
- Distributed tracing end-to-end
- Metrics: latency percentiles, error rates, throughput, saturation
- Alerting linked to runbooks
- Emergency rollback mechanism always available

## Deployment Safety
- Phased deployments with canary
- Backward/forward compatible schema changes (two-phase)
- Feature flags for gradual rollout
- Automatic rollback on metric breach
- Deploy during business hours only
