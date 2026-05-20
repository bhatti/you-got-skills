---
name: ygs-code-review
description: Diff-based code review — two-pass (critical/informational), testing discipline, fix-first pattern. Use before committing or on a PR branch. Best with reasoning model (Opus-class).
---

# Code Review

For detailed principles, read:
- `references/code-quality.md` — Cognitive load, tidying, testing
- `references/functional-design.md` — Immutability, FSM, making invalid states impossible

## Step 1: Get the diff

```bash
git diff --stat
git diff main...HEAD --stat 2>/dev/null || git diff master...HEAD --stat 2>/dev/null
```

If a PR number is given and `gh` is available:

```bash
gh pr diff <number>
```

## Step 2: Read changed files in full context

Read the full diff AND surrounding code (not just changed lines).

## Step 3: Pass 1 — Critical issues (block merge)

1. **Correctness** — Logic errors, off-by-one, null handling, race conditions
2. **Security** — Injection, XSS, SSRF, path traversal, hardcoded secrets
3. **Data loss** — Destructive operations without confirmation, missing transactions
4. **Race conditions** — TOCTOU, check-then-act, find-or-create without locks
5. **Error swallowing** — Empty catch blocks, ignored return values, silent failures
6. **Enum completeness** — New enum values traced through ALL consumers
7. **Partial failure** — What if operation half-succeeds? Inconsistent state possible?

## Step 4: Pass 2 — Design & maintainability

1. **Testability** — Can this be tested without mocking internal code? If not, design is wrong.
2. **Testing quality** — Tests verify public contract? Meaningful assertions (not just coverage)?
3. **Immutability & state** — Is mutable state minimized? Are invalid states representable? Should this use a state machine?
4. **Type safety** — Are sum types / enums used for variants? Newtypes for distinct IDs? Parse-don't-validate at boundaries?
5. **Naming** — Clear, consistent, intention-revealing
6. **Complexity** — Could be simpler? Unnecessary abstractions? Justified with data?
7. **Duplication** — Existing code already does this?
8. **Interface design** — Proper DI? Deep modules (small interface, rich implementation)? CQS respected?
9. **Performance** — N+1 queries, missing indexes, O(n²) in loops, unbounded allocations
10. **Error handling** — Errors as values (not exceptions for control flow)? Propagated with context?
11. **Boundary handling** — Empty inputs, max limits, type coercion at system boundaries
12. **Dead code** — Unreachable paths, unused imports, stale comments

## Step 5: Testing discipline check

- Are changes covered by tests?
- Do tests use real implementations for owned code (not mocks)?
- Stubs only at 3rd-party/OS boundaries (HTTP clients, clocks, filesystem)?
- Do tests verify behavior through public API, not internal state?
- Filesystem access isolated? (no shared test directories that could conflict)

## Step 6: Proportionality check

- Is the change proportional to the problem?
- Over-engineered for the use case? (abstraction without multiple consumers)
- Under-engineered? (skipped error handling on critical path)

## Step 7: Fix-first pattern

For obvious mechanical issues (typos, unused imports, formatting):
- Fix them directly if user asked for fixes
- Otherwise note as "auto-fixable"

For judgment calls (architecture, naming, approach):
- Present concern + suggested alternative
- Mark as ASK — don't auto-fix

## Step 8: Report

For each finding:
- **Severity:** MUST (blocks merge) | SHOULD (strong recommendation) | MAY (suggestion)
- **Confidence:** HIGH (certain) | MEDIUM (likely) | LOW (preference)
- **File:line** reference
- **Issue** and suggested fix

Summary verdict:
- **Approve** — Ship it
- **Request changes** — Fix MUST items before merge
- **Comment** — Suggestions only, no blockers

Report **DONE** or **DONE_WITH_CONCERNS**.
