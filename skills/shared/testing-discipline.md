# Testing Discipline

Canonical testing rules for all skills that write, review, or assess tests.
Reference this file — do not inline these rules.

## Rules

**TDD is mandatory.** Write a failing test before implementing. Red → green → refactor — in that order, every time. Never write implementation code before there is a test that requires it.

**Tests use production code paths.** Tests must call the same methods, constructors, and code paths that production calls. Never add test-only parameters, flags, or dependency injection points specifically to enable testing — that's a design smell, not a testing strategy.

**Coverage gate: 90%+.** Every new code path must be covered. Fail CI if coverage drops below threshold. But coverage is a floor, not a goal — 90% with weak assertions is worse than 80% with strong ones.

**No flaky tests.** A test that passes sometimes and fails other times is worse than no test — it erodes trust in the suite. Root causes: timing, shared mutable state, network calls, filesystem ordering, random data without seeds.

**No sleeps.** Never use `sleep()`, `Thread.sleep()`, `time.sleep()` or equivalent in tests. Use condition variables, polling with timeout, or mock time instead.

**No mocking internal code.** Only mock at system boundaries (external HTTP, databases, queues, clocks). Mocking your own functions creates tests that pass even when the implementation is broken.

**Deterministic time.** Inject clocks — never call `Date.now()`, `time.time()`, `System.currentTimeMillis()` directly in testable code. Tests that depend on wall-clock time are implicitly flaky.

**No shared mutable state between tests.** Each test must set up and tear down its own state. Global/static state that bleeds between tests causes order-dependent failures.

**Test names describe behavior.** "should return 404 when user not found" beats "test_get_user_3". A failing test name should tell you what broke without reading the body.

**One assertion per concept.** Multiple independent assertions in one test body make it hard to identify which one failed. Split them if they test different behaviors.

**Real method calls, not mocks, for the unit under test.** The code you're testing must actually run — not be replaced by a stub. A test that stubs its own subject is testing nothing.

**Passing tests ≠ correct behavior.** After tests pass, ask: could these pass even if the implementation were wrong? Overly loose assertions (checking that a value is non-null rather than correct) create false confidence.

## Red flags to report

- Empty test bodies or `// TODO: add assertions`
- Tests that always pass (no assertions, assertion in wrong place)
- Tests named after implementation details, not behaviors
- Test file with zero coverage of the changed path
- Setup/teardown that creates side effects visible to other tests
- Unused test variables, dead test helper functions
- Test-only flags or constructors added to production code to enable testing

## Iron Law

**Code written before tests must be deleted and rewritten test-first.** This is not negotiable. The discipline breaks the moment you allow "just this once" exceptions. Pre-written code is written to pass, not to be tested — it produces tests that verify the implementation was typed correctly, not that it works.

## Common Rationalizations

| Rationalization | Reality |
|----------------|---------|
| "The code is so simple it doesn't need a test" | Simple code breaks in simple ways. The test documents the expected behavior and catches regressions you forgot about in 6 months. |
| "I'll add tests after to save time" | Code written without tests is written to be untestable. You will spend more time retrofitting tests than writing them first. |
| "This is just a prototype" | Prototypes become production. The test you skip today is the regression you chase at 2am. |
| "The test would just mirror the implementation" | A test that mirrors the implementation tests that you typed it correctly, not that it works. Name the observable behavior, not the implementation steps. |
| "We're under time pressure" | Under pressure is when bugs are most expensive. Tests are insurance that pays off immediately when you change related code. |
| "I can't test this because of external dependencies" | You can test the logic with fakes at the boundary. If you can't isolate the logic, that's a design problem — fix the design. |
| "The test runner is slow" | Fix the test runner. Do not skip tests because running them is inconvenient. A slow test suite is a CI problem to fix, not a testing discipline to abandon. |

## Async and timing scenarios

For tests that wait for async conditions: see `~/.claude/skills/you-got-skills/skills/shared/condition-based-waiting.md` — replace sleep with condition polling.
