# Production Readiness Checklist

## Build & Deploy
- Separate CI/CD pipelines for code and infrastructure
- Vulnerable dependencies identified during build
- Phased deployments with canary testing
- Automatic rollback on test failure or metric breach
- Deploy during business hours, not during outages/peaks
- Immutable infrastructure (no in-place patches)

## Resilience Patterns
- **Circuit breaker** — Stop requests to failing services
- **Retry with jitter** — Exponential backoff + randomization to prevent thundering herd
- **Bulkhead isolation** — Thread/connection pool separation per dependency
- **Graceful degradation** — Partial functionality during dependency failure
- **Graceful shutdown** — Drain in-flight requests on SIGTERM
- **Health checks** — Deep checks (DB, cache, dependencies), not just "200 OK"
- **Rate limiting** — Per-account and per-API with proper 429 responses
- **Idempotency** — Safe to retry all mutating operations

## Fault Tolerance
- No hard startup dependencies (start even if deps unavailable)
- Single points of failure eliminated
- Redundancy and automatic failover configured
- Data redundancy with tested backup/recovery
- Cold-start times optimized

## Observability
- **Metrics** — Latency percentiles (p50, p95, p99), error rates, throughput, saturation
- **Logging** — Structured with correlation IDs, proper levels, no PII
- **Tracing** — Distributed tracing end-to-end with sampling
- **Alerting** — Threshold-based + anomaly detection, linked to runbooks
- **Dashboards** — Real-time and historical, covering SLO compliance

## Transaction Safety
- Transaction boundaries explicitly defined
- Dual-write problem addressed (outbox pattern or CDC)
- Optimistic concurrency for low-contention paths
- SAGA pattern for distributed transactions with compensation
- Partial failure handling (no inconsistent state)

## Capacity & Performance
- Load tested with real-world scenarios and varied data
- Autoscaling policies validated under load
- Connection pools sized correctly (> thread count + buffer)
- Pagination on all unbounded queries (with expiring tokens)
- Caching strategy validated (TTL, invalidation, pre-warming)
- Database indexes reviewed for query patterns

## Data Management
- Schema changes backward/forward compatible (two-phase releases)
- Data retention policies established and enforced
- Backup/recovery tested against RPO and RTO targets
- Data migration tested in isolated environments
- Stale data and log cleanup automated

## Disaster Recovery
- Failover procedures documented and tested
- Chaos engineering practiced regularly
- Emergency rollback mechanism available ("big red button")
- Communication channels tested (don't rely on affected systems)
- Geographic redundancy for critical services

## Configuration & Feature Flags
- Config in source control with deployment pipeline
- Feature flags managed centrally with testing
- All flag combinations tested
- Config validated in test before production
