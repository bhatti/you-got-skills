---
name: ygs-code-review
description: Diff-based code review — two-pass (critical/informational), testing discipline, fix-first pattern. Use before committing or on a PR branch. Best with reasoning model (Opus-class).
---

# Code Review

For shared review protocol (diff, severity, output format), read `~/.claude/skills/you-got-skills/skills/shared/review-scaffold.md`.

For detailed principles, read:
- `references/code-quality.md` — Cognitive load, tidying, testing
- `references/functional-design.md` — Immutability, FSM, making invalid states impossible
- `~/.claude/skills/you-got-skills/skills/shared/testing-discipline.md` — testing rules (referenced in Step 6)
- `~/.claude/skills/you-got-skills/skills/shared/functional-design.md` — functional design checklist (referenced in Step 5)

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

## Step 4: Pass 1 — Critical and Architecture issues (block merge)

0. **Architecture impact** — Does this change introduce architectural debt or break module boundaries?
   - New circular dependency introduced
   - Dependency flowing in the wrong direction (e.g., domain layer importing from infrastructure)
   - Pattern introduced without justification or existing precedent in the codebase
   - Abstraction added for a single consumer (one consumer = wrapper, not abstraction)
   - Alignment with existing norms violated — would this surprise a reader familiar with the codebase?
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
2. **Testability** — Can this be tested without mocking internal code? If not, design is wrong.
3. **Testing quality** — Tests verify public contract via real method calls? Assertions are data-driven (no hunches)?
4. **Immutability & state** — Mutable state minimized? Invalid states representable? State machine appropriate?
   Reference: `~/.claude/skills/you-got-skills/skills/shared/functional-design.md` for the full checklist.
5. **Type safety** — Sum types / enums for variants? Parse-don't-validate at boundaries?
6. **Naming** — Clear, consistent, intention-revealing; no underscores for private methods
7. **Complexity** — Could it be simpler? Unnecessary abstractions? Proportional to the problem?
8. **Duplication** — Does existing code already do this?
9. **Interface design** — Deep modules (small interface, rich implementation)? CQS respected? Encapsulation intact?
10. **Performance** — N+1 queries, missing indexes, O(n²) in loops, unbounded allocations
11. **Error handling** — Errors as values? Propagated with context? No interpolation in log messages?
12. **Boundary handling** — Empty inputs, max limits, coercion at system boundaries
13. **Dead code** — Unused imports, variables, stale comments, empty test bodies
14. **Circular dependencies** — Are any new import cycles introduced?
15. **Scope hygiene** — Changes unrelated to the task? Issue numbers in comments/code?

## Step 6: Testing discipline check

Follow `~/.claude/skills/you-got-skills/skills/shared/testing-discipline.md` for the full testing discipline checklist.

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

---

## Common Rationalizations

| Rationalization | Reality |
|----------------|---------|
| "I'm confident enough to skip Pass 0" | Confidence correlates poorly with correctness. Run Pass 0 first — if the approach is wrong, individual code issues are irrelevant. |
| "This finding is minor, I won't flag it" | Minor findings compound. Flag at MAY severity and let the author decide — your job is to surface, not filter. |
| "The existing code does it this way so it must be right" | Normalization of deviance: existing patterns can be wrong. Flag it regardless of precedent. |
| "I'll check the architecture axis later in Pass 2" | Architecture debt that slips through Pass 1 gets merged. It's in Pass 1 for a reason. |
| "I can't find an obvious bug, so the code is good" | Absence of obvious bugs is not a green flag. Check the architecture, testability, and error paths. |
