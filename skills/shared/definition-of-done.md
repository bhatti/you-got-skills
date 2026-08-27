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

- [ ] No debug artifacts: no `console.log` left in, no `TODO: remove`, no hardcoded test values
- [ ] No dead code introduced: no unused imports, variables, empty test bodies
- [ ] No issue/ticket numbers in code or comments — explain the *why*, not the ticket
- [ ] No circular dependencies introduced
- [ ] Naming is consistent with surrounding code; no surprises for a reader

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
