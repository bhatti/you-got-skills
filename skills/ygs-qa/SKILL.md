---
name: ygs-qa
description: QA testing — functional testing, edge cases, regression checks, and health scoring. Use when code is ready for quality validation.
---

# QA Testing

For reliability testing patterns, read `~/.claude/skills/you-got-skills/skills/ygs-sre-review/references/testing-reliability.md`.

## Step 1: Understand scope

Determine what changed:

```bash
git diff main...HEAD --stat 2>/dev/null || git diff master...HEAD --stat 2>/dev/null
```

Read relevant task files or PRD for acceptance criteria:

```bash
ls tasks/in-progress/*.md tasks/done/*.md 2>/dev/null | head -5
```

## Step 2: Build test matrix

From the changes and acceptance criteria:

### By severity
- **Critical** — Data loss, crashes, security holes (must fix)
- **High** — Feature unusable, blocking workflow (must fix)
- **Medium** — Workaround exists but degraded experience (should fix)
- **Low** — Cosmetic, polish (nice to fix)

### By category
- **Functional** — Does core logic work correctly?
- **Edge cases** — Boundary values, empty inputs, max limits, null handling
- **Error paths** — Invalid input, network failures, timeouts, partial failures
- **Regressions** — Do existing features still work?
- **Performance** — Acceptable response times? No degradation?
- **Accessibility** — Keyboard nav, screen readers, contrast (if UI)
- **Console/Errors** — Clean console? No unhandled exceptions?

## Step 3: Run existing tests

Use the polyglot test runner from `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.

## Step 4: Identify coverage gaps

For each acceptance criterion: is there a test that verifies it?
For each new code path: is there a unit test covering it?
For each new public API/export: is there a test exercising it?

Report specifics — file paths, line numbers, criterion text. Not generalities.

## Step 5: Verify scenarios

For each test scenario:
- Trace the code path
- Check state consistency after failures
- Verify error handling is correct and user-friendly
- Check for silent failures (swallowed exceptions, partial completions)

## Step 6: Write missing tests

If gaps are found, write tests for uncovered critical/high scenarios. Max 3 QA rounds — if still failing after 3, report status and let user decide.

## Step 7: Health score report

### Test results

| Suite | Tests | Pass | Fail | Skip |
|-------|-------|------|------|------|

### Coverage (modified files)

| File | Lines | Branches | Uncovered |
|------|-------|----------|-----------|

Rate each category 0-10:

| Category | Score | Issues |
|----------|-------|--------|
| Functional | /10 | |
| Error handling | /10 | |
| Edge cases | /10 | |
| Performance | /10 | |
| Regressions | /10 | |

**Overall health: X/50**

### Issues found
For each issue:
- Severity + category
- File:line reference
- Expected vs actual behavior
- Suggested fix

### Ship readiness
- **Ready** — No critical/high issues remaining
- **Not ready** — List blockers
- **Ready with concerns** — Shippable but flagging risks

Report **DONE**, **DONE_WITH_CONCERNS**, or **BLOCKED**.
