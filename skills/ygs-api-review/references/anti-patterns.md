# API Anti-Patterns Checklist

Check for these common API mistakes during review:

## Design
- Bottom-up design (implementation first, spec later)
- Bloated API surface (too many endpoints tailored to UI)
- Verbs in URLs, inconsistent resource naming
- Execute anti-pattern (one endpoint with conditional branching)
- No clear internal/external API boundary
- Mixing control-plane and data-plane in same service

## Contract & Consistency
- Inconsistent naming across services
- Wrong HTTP verb for operation (POST for reads, GET for mutations)
- Breaking changes without versioning
- Leaky abstractions (DB internals in errors, readable pagination tokens)
- Missing or inconsistent input validation
- Bimodal behavior (different response shapes under load)

## Efficiency
- N+1 queries / ORM abuse
- Missing pagination on list endpoints
- Chatty APIs requiring multiple sequential calls
- Synchronous APIs for long-running operations (no job ID, no status endpoint)
- Batch operations with mixed success/error in flat list

## Idempotency & Transactions
- Missing idempotency tokens on create operations
- Transaction boundary violations (multi-resource update without atomicity)
- PATCH that deletes fields not included in request
- Missing optimistic concurrency control (last write wins)
- Dual-write problem (DB + event without outbox pattern)

## Error Handling
- Opaque, non-actionable errors ("Something went wrong")
- Error messages that clients must parse (info embedded in strings)
- Internal information leaked in errors (stack traces, SQL, hostnames)

## Resilience
- No retry safety (retrying non-idempotent operations)
- Retry storms without jitter
- Hard startup dependencies (won't start if dependency down)
- Missing graceful shutdown (drops in-flight requests on SIGTERM)
- Shallow health checks (returns 200 while DB is exhausted)
- No emergency rollback mechanism

## Security & Lifecycle
- Missing runtime boundary validation (OpenAPI not enforced)
- PII data exposure in responses/logs
- No rate limiting or per-account quotas
- Cache keys ignoring authorization
- Deprecated endpoints without migration path or sunset timeline
- No API versioning strategy
