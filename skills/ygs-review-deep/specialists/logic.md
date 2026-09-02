# Specialist: Logic & Correctness

Review scope: does the code do what it claims? Are there failure paths the author did not consider?

## Checks

**Logic errors**
- Off-by-one in loops and slices
- Null/nil dereference without guard
- Integer overflow or underflow on arithmetic
- Wrong operator precedence or short-circuit evaluation assumption

**Error path handling**
- Every error is propagated or explicitly handled — no silent ignoring
- Error messages include context (which operation, which input, which ID)
- No swallowed errors in catch blocks, `.unwrap()` without justification, `? `discarded

**Partial failure**
- If step N of M fails, what is the system state? Can it be retried safely?
- Idempotency: does re-running the operation produce correct results?
- Are transactions scoped correctly — no partial commits visible to readers

**Edge cases**
- Empty inputs (empty string, empty list, zero)
- Maximum-size inputs (large payloads, max integer)
- Concurrent duplicate operations (create-or-find races)
- Time boundary conditions (midnight, end-of-month, timezone edges)

**Control flow**
- Unreachable branches that could actually be reached
- Missing `else` branches that leave variables uninitialized
- Early returns that skip cleanup (deferred calls, lock releases)

## Severity guidance

| Severity | Example |
|----------|---------|
| MUST | Logic error that produces wrong results or data corruption |
| MUST | Swallowed error on a critical path |
| SHOULD | Missing edge case test for a boundary condition |
| MAY | Stylistic improvement to error message content |
