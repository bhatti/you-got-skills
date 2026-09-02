# Definition of Done

The Definition of Done (DoD) is the standing quality bar every change must clear before it is DONE — not just "code written." It applies at three scopes:

| Scope | Applies To | Full DoD Sections Required |
|-------|-----------|---------------------------|
| **Per task** | Each implementation task | Correctness + Quality |
| **Per feature** | A shippable user-facing feature | + Integration + Documentation |
| **Per release** | A versioned release or PR | All four sections |

Use this alongside `completion-signals.md` — the DoD determines whether DONE or DONE_WITH_CONCERNS applies. If any line below isn't met and can't be deferred, the status is DONE_WITH_CONCERNS (document the gap) or BLOCKED (gap prevents safe shipping).

---

## 1. Correctness

- [ ] The acceptance criteria are satisfied — traced through the code path, not just asserted in prose
- [ ] All tests pass (run the full suite via `~/.claude/skills/you-got-skills/skills/shared/test-runner.md` — not just the affected subset)
- [ ] New code paths have test coverage: happy path, error path, and at least one boundary condition
- [ ] No regression in existing passing tests
- [ ] The actual behavior was exercised end-to-end, not just unit-tested (CLI run, HTTP call, browser smoke)

## 2. Quality

- [ ] No debug artifacts: no `console.log` left in, no `TODO: remove`, no hardcoded test values — all debug/trace logs gated by log level
- [ ] No dead code introduced: no unused imports, variables, empty test bodies, assertion-free tests, dead test helpers
- [ ] No legacy or backward-compatibility code — delete cleanly; no removed/moved comments
- [ ] No issue/ticket numbers in code or comments — explain the *why*, not the ticket
- [ ] No circular dependencies introduced; cyclic module refs resolved properly, not via trait hacks
- [ ] Naming is consistent with surrounding code; no surprises for a reader; no leading-underscore private methods
- [ ] Tests use the same code paths as production — no test-only flags, parameters, or dependency injections
- [ ] 90%+ coverage gate — every new code path has test coverage
- [ ] `docs/*.md` and README/examples updated when interfaces or behavior change
- [ ] SDK/wrappers are decorators only — core logic stays in the underlying framework, not the wrapper
- [ ] Deep modules: hide complexity behind a small, stable interface; no shallow pass-through layers

## 3. Integration

- [ ] Build passes cleanly (no warnings treated as errors, no suppressed lints)
- [ ] No new external dependencies added without justification and pinned version
- [ ] Schema migrations (if any) are backwards-compatible or coordinated with deployment
- [ ] Feature flags (if any) are wired and tested in both the on and off state

## 4. Ship-readiness

- [ ] Security implications reviewed: auth check, input validation, no secrets in code
- [ ] Observability for new critical paths: at minimum, failures are logged with context
- [ ] Rollback path exists: can this be reverted without data loss or manual intervention?
- [ ] No `TODO: before ship` markers remain

---

## Red Flags

- "Tests pass" used as a synonym for done — tests verify what was tested, not all behaviors
- Applying a looser DoD under deadline pressure — the risk doesn't decrease, the detection does
- Skipping the exercise step (Step 9 in ygs-implement) — most production bugs survive tests but fail on first use
