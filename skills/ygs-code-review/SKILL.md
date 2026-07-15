---
name: ygs-code-review
description: Diff-based code review — two-pass (critical/informational), testing discipline, fix-first pattern. Use before committing or on a PR branch. Best with reasoning model (Opus-class).
---

# Code Review

For shared review protocol (diff, severity, output format), read `~/.claude/skills/you-got-skills/skills/shared/review-scaffold.md`.

For detailed principles, read:
- `references/code-quality.md` — Cognitive load, tidying, testing
- `references/functional-design.md` — Immutability, FSM, making invalid states impossible

## Step 1: Get the diff

Follow the diff protocol from `shared/review-scaffold.md`.

## Step 2: Read changed files in full context

Read the full diff AND surrounding code (not just changed lines).

## Step 3: Pass 0 — Is the approach right?

Before checking code quality, assess whether the change is solving the right problem the right way:
- Does this address the root cause, or paper over a symptom?
- Is there a materially simpler way to achieve the same goal?
- Does the approach conflict with how the system actually works? (Check surrounding code, not just the diff)
- Is the scope proportional to the problem? (Over-engineered? Under-engineered?)

If the approach is fundamentally wrong, that's the finding — individual code issues are irrelevant if the direction is bad.

## Step 4: Pass 1 — Critical issues (block merge)

1. **Correctness** — Logic errors, off-by-one, null handling, race conditions
2. **Security** — Injection, XSS, SSRF, path traversal, hardcoded secrets, ungated debug logs
3. **Data loss** — Destructive operations without confirmation, missing transactions
4. **Race conditions** — TOCTOU, check-then-act, find-or-create without locks; concurrent access at scale
5. **Error swallowing** — Empty catch blocks, ignored return values, silent failures
6. **Enum completeness** — New enum values traced through ALL consumers
7. **Partial failure** — What if operation half-succeeds? Inconsistent state possible?
8. **Scalability** — Unbounded allocations, N+1 patterns, behavior under millions of requests or large data

## Step 5: Pass 2 — Design & maintainability

1. **Correctness of feedback** — Double-check every finding against the actual diff; no false positives
2. **Alignment with existing norms** — Does the change match existing patterns, conventions, and architecture? Would it surprise a reader?
3. **Testability** — Can this be tested without mocking internal code? If not, design is wrong.
4. **Testing quality** — Tests verify public contract via real method calls? Assertions are data-driven (no hunches)?
5. **Immutability & state** — Mutable state minimized? Invalid states representable? State machine appropriate?
6. **Type safety** — Sum types / enums for variants? Parse-don't-validate at boundaries?
7. **Naming** — Clear, consistent, intention-revealing; no underscores for private methods
8. **Complexity** — Could it be simpler? Unnecessary abstractions? Proportional to the problem?
9. **Duplication** — Does existing code already do this?
10. **Interface design** — Deep modules (small interface, rich implementation)? CQS respected? Encapsulation intact?
11. **Performance** — N+1 queries, missing indexes, O(n²) in loops, unbounded allocations
12. **Error handling** — Errors as values? Propagated with context? No interpolation in log messages?
13. **Boundary handling** — Empty inputs, max limits, coercion at system boundaries
14. **Dead code** — Unused imports, variables, stale comments, empty test bodies
15. **Circular dependencies** — Are any new import cycles introduced?
16. **Scope hygiene** — Changes unrelated to the task? Issue numbers in comments/code?

## Step 6: Testing discipline check

- Are changes covered by tests?
- Do tests call real methods and assert observable side effects, not implementation details?
- Stubs only at 3rd-party/OS boundaries (HTTP clients, clocks, filesystem)?
- Use deterministic time controls (fake clocks) — not real sleeps or timers
- Tests must not be flaky: no timing-sensitivity, no shared mutable state, no concurrency races
- No empty test bodies, no unused variables, no dead code in tests
- Existing tests still pass — do not break what wasn't broken

## Step 7: Proportionality check

- Is the change proportional to the problem?
- Over-engineered for the use case? (abstraction without multiple consumers)
- Under-engineered? (skipped error handling on critical path)

## Step 8: Fix-first pattern

For obvious mechanical issues (typos, unused imports, formatting):
- Fix them directly if user asked for fixes
- Otherwise note as "auto-fixable"

For judgment calls (architecture, naming, approach):
- Present concern + suggested alternative with code references (file:line)
- Mark as ASK — don't auto-fix
- Be specific — vague feedback wastes reviewer time

**Feedback quality gate:** Before reporting, verify each finding is correct against the actual code. No hallucinations. No "I think this might be a problem." If uncertain, say so explicitly.

## Step 9: Report

Use the finding format and verdict from `shared/review-scaffold.md`.

Report **DONE** or **DONE_WITH_CONCERNS**.
