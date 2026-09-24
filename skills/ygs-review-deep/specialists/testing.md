# Specialist: Testing Quality

Review scope: do the tests actually verify what the code claims to do?

For canonical testing rules, read `~/.claude/skills/you-got-skills/skills/shared/testing-discipline.md`.

## Coverage of new code

- Every new code path has at least one test: happy path, error path, boundary condition
- New security or data-integrity paths have tests before the code is considered done
- A diff with no test changes and non-trivial new logic is a red flag

## Test design

- Tests verify behavior (what the code does externally), not implementation (how it does it internally)
- **Prefer actual method calls:** tests should call the same methods production calls and assert on observable side effects — mock counts for internally-owned code prove nothing; mocks belong only at system boundaries (external HTTP, DB, filesystem, clock, OS)
- Tests coupled to private methods or internal state break on safe refactors — flag as SHOULD
- **No test-only production params:** production code must not carry flags, parameters, or injection points added solely for test convenience — if something is hard to test, that is a design signal, not a reason to widen the production interface

## Test validity

- Empty test bodies or tests with no assertions always pass regardless of code changes
- Assertions check actual correctness, not just non-null / non-zero: `assert_eq!(result, expected)` not `assert!(result.is_some())`

## Flaky pattern prevention

- **`time.sleep()` / `Thread.sleep()` in tests is a MUST-level finding** — use condition variables, channels, event-based synchronization, or deterministic sequencing instead; timing-based tests are unreliable under load and in CI
- Shared mutable state between tests that can bleed across test runs — use fresh instances per test
- Fixed ports, file paths, or process IDs that conflict in parallel runs — use ephemeral/random values
- Non-deterministic ordering of results without explicit sorting or set comparison
- Concurrent test setup that races between tests — use proper barriers or sequential setup

## Edge case coverage

- Boundary inputs: empty list, single element, maximum size, zero values, nil/null
- Concurrent access: does the test exercise concurrent usage for code that will run concurrently?
- Idempotency: is re-running the operation tested?

## Test simplicity

- **Concise comments:** test comments should state the WHY (non-obvious constraint or invariant), not restate WHAT the assertion checks; one short line max
- **Don't use grep to validate tests:** when assessing whether a test exists or passes, run the test suite — do not use grep as evidence of coverage; grep finds declarations, not execution
- **Keep tests simple and reliable:** a test that requires complex setup or deep knowledge to read is harder to trust; prefer direct, flat test structure
- Tests should fail clearly and immediately when the behavior they cover breaks — not silently pass with a wrong value

## Don't touch unrelated code

- A diff that modifies comments, formatting, logic, or test cases outside the stated change is scope creep — flag as SHOULD with a question asking if the change is intentional
- Don't change log messages, error messages, or existing comments needlessly — downstream systems may have regex keyed on them, and unnecessary churn obscures the real change

## Dead test code

- Unused test helper functions that are never called
- Unused test variables or setup code that doesn't affect any assertion
- Commented-out test bodies

## Severity guidance

| Severity | Example |
|----------|---------|
| MUST | New security or data-integrity code with no test |
| MUST | Test that always passes regardless of implementation (no assertions) |
| MUST | `sleep`-based synchronization introduced in a test |
| SHOULD | Entire new feature path with no integration test |
| SHOULD | Mock of internally-owned code replacing an actual method call |
| SHOULD | Production method modified to accept test-only flag/param |
| SHOULD | Flaky pattern introduced (shared state, fixed ports, non-deterministic ordering) |
| MAY | Missing edge case on a low-risk path |
| MAY | Test comment that restates what the code does rather than why |
