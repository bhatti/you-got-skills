# API Design Patterns

Positive patterns to look for and recommend during review:

## Resource Design
- **Processing Resource** — Triggers actions with clear contract
- **Information Holder** — Exposes domain data (operational, master, reference)
- **Computation Function** — Side-effect-free processing based on input only

## Operations
- **State Creation** — Fire-and-forget or deferred processing with clear semantics
- **Retrieval** — Read-only access enabling client processing
- **State Transition** — Provider-side state change with ACID guarantees
- **Command-Query Separation** — Isolate mutations from queries

## Message Structure
- **Embedded Entity** — Inline related data to reduce roundtrips
- **Linked Information Holder** — Reference related data to keep messages small
- **Error Report** — Machine-readable codes with human-readable messages

## Client-Driven Content
- **Pagination** — Page-based, cursor-based, or offset-based (with expiring tokens)
- **Wish List** — Client specifies desired fields (sparse fieldsets)
- **Conditional Request** — Execute only when metadata conditions met (ETags)
- **Request Bundle** — Multiple independent operations in single request

## Evolution
- **Semantic Versioning** — x.y.z communicating compatibility
- **Two in Production** — Maintain two versions for gradual migration
- **Limited Lifetime Guarantee** — Communicate supported timeframes
- **Aggressive Obsolescence** — Release → Deprecate → Decommission lifecycle

## Resilience Patterns
- **Idempotency tokens** — Client-provided key for safe retries on create
- **Outbox pattern** — Events stored in same DB transaction, relayed async
- **Circuit breaker** — Stop requests to failing downstream
- **Retry with jitter** — Exponential backoff + randomization
- **Graceful degradation** — Partial functionality during dependency failure
