# AI Failure Modes

Language-agnostic catalogue of patterns where AI-generated code characteristically fails review. Used for self-audit during implementation and as a specialist check during review.

**For each pattern: verify the assertion would fail if the behavior it covers actually broke.**

---

## The 8 Patterns

### 1. Type Escape Hatch
Using the language's "any" equivalent (`any`, `interface{}`, `object`, `dynamic`, `void*`) as a shortcut when types are complex. This makes the compiler blind to actual type errors downstream.

**Red flag:** A type escape immediately adjacent to a complex data structure or return value.
**Fix:** Model the actual type, or use `unknown`/`any` with explicit narrowing before use — never unguarded.

---

### 2. Hollow Tests
Assertions that cannot fail regardless of the behavior under test:
- Asserting `.toBeDefined()` / `!= null` / `is not None` when the code always returns a value
- Mocking the exact thing under test, then asserting the mock was called
- Single call-count assertions with no state or output verification
- Test setup that overrides the production code path being tested

**Verification rule:** For every assertion, ask: "what code change would make this assertion fail?" If you can't name one, the test is hollow.

---

### 3. Sequential Async Where Parallel Is Safe
Awaiting inside a loop when results are independent:
```
for item in items:
    await process(item)   # ← N serial round trips
```
Should be concurrent dispatch: `Promise.all`, `asyncio.gather`, goroutine + channel, `pmap`, etc.

**Exceptions:** (1) When each iteration's output feeds the next (data pipeline), sequential is correct. (2) When operations have ordered side effects that must be applied in sequence (e.g., write-ahead log entries, state machine transitions). Verify the dependency or ordering constraint before parallelizing.

---

### 4. Swallowed Errors
```
try:
    risky_operation()
except Exception as e:
    logger.error(f"Failed: {e}")
    # execution continues in broken state
```
The error is acknowledged but the caller receives no indication of failure. Downstream code proceeds with potentially invalid state.

**Fix:** Either propagate the error (re-raise, return Result/Either, surface to caller) or explicitly document why silent continuation is correct in this case.

---

### 5. Generic Log Messages Without Correlation IDs
```
logger.error("Failed to process record")       # ← impossible to grep in prod
logger.error(f"Failed to process record: {e}") # ← slightly better, still useless
```
Without entity IDs (user ID, request ID, record ID, job ID), log lines cannot be correlated to a specific failure in production.

**Fix:** Include at minimum: operation name + entity identifier + error detail. Example: `logger.error("process_record failed", record_id=id, error=str(e))`

---

### 6. Incomplete Variant Coverage
A new enum value, discriminated union variant, or tagged union case is handled in one switch/match/dispatch but not in all consuming handlers in the codebase.

**Check:** For every new variant, grep for all existing switches on that type. Each must have an explicit case or a documented default that handles the new value correctly.

---

### 7. Format-Valid But Semantically Invalid Inputs
Validates that input matches a pattern or type, but not that it is within a safe or meaningful range:
- URL validated as `string` but not constrained to safe origins (SSRF risk)
- Integer validated as `int` but negative values not excluded for a count field
- String validated as non-empty but `"  "` (whitespace) accepted as an identifier

**Fix:** Parse, don't validate — transform input at the boundary into a constrained type that makes invalid values unrepresentable.

---

### 8. Parallel Flow Written as Sequential
Independent operations with no data dependency written in series, multiplying latency by count:
```
result_a = fetch_a()   # waits
result_b = fetch_b()   # waits for no reason — A and B are independent
result_c = fetch_c()   # waits again
```
When A, B, and C have no dependency, they should be dispatched concurrently.

**Exception:** When operations share a resource with limited concurrency (DB connection pool, rate-limited API), controlled serial execution may be intentional. Verify before flagging.

---

## How to Apply

**During implement (Step 7 — Write tests, Step 10 — Self-review):**
Read through your own diff and check against each of the 8 patterns. AI agents are most susceptible to patterns 2 (hollow tests), 3 (sequential async), and 4 (swallowed errors).

**During review:**
Flag each confirmed match as a finding. Patterns 1, 4, and 6 are typically MUST-level. Patterns 2, 3, 7, 8 are typically SHOULD-level. Pattern 5 is typically SHOULD.

All patterns: verify against the actual diff before reporting — no hallucinations.
