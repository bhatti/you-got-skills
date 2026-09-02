# Specialist: Testing Quality

Review scope: do the tests actually verify what the code claims to do?

For canonical testing rules, read `~/.claude/skills/you-got-skills/skills/shared/testing-discipline.md`.

## Coverage of new code

- Every new code path has at least one test: happy path, error path, boundary condition
- New security or data-integrity paths have tests before the code is considered done
- A diff with no test changes and non-trivial new logic is a red flag

## Test design

- Tests verify behavior (what the code does externally), not implementation (how it does it internally)
- Tests call the same methods production calls — no test-only parameters or flags on production code
- Tests coupled to private methods or internal state break on safe refactors

## Test validity

- Empty test bodies or tests with no assertions always pass regardless of code changes
- Assertions check actual correctness, not just non-null / non-zero: `assert_eq!(result, expected)` not `assert!(result.is_some())`
- Mocks only at system boundaries (external HTTP, DB, clock, OS); no mocking of internally-owned code

## Flaky patterns

- `time.sleep()` / `Thread.sleep()` in tests — use condition variables or event-based synchronization
- Shared mutable state between tests that can bleed across test runs
- Fixed ports, file paths, or process IDs that conflict in parallel runs
- Non-deterministic ordering of results without explicit sorting or set comparison

## Edge case coverage

- Boundary inputs: empty list, single element, maximum size, zero values, nil/null
- Concurrent access: does the test exercise concurrent usage for code that will run concurrently?
- Idempotency: is re-running the operation tested?

## Dead test code

- Unused test helper functions that are never called
- Unused test variables or setup code that doesn't affect any assertion
- Commented-out test bodies

## Severity guidance

| Severity | Example |
|----------|---------|
| MUST | New security or data-integrity code with no test |
| MUST | Test that always passes regardless of implementation (no assertions) |
| SHOULD | Entire new feature path with no integration test |
| SHOULD | Flaky pattern introduced (`sleep`, shared state) |
| MAY | Missing edge case on a low-risk path |
