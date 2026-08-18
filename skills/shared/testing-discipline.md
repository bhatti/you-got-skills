# Testing Discipline

Canonical testing rules for all skills that write, review, or assess tests.
Reference this file — do not inline these rules.

## Rules

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
