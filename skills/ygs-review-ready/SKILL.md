---
name: ygs-review-ready
description: >
  Use before opening a PR, from a pre-push hook, or in CI to get a single pass/fail
  readiness signal. Runs hygiene, tests, and a quick review pass in one shot. Lighter
  than ygs-ship — no version bump, no PR creation.
argument-hint: "[--base <branch>] [--head <ref>] [--skip-ci] [--skip-review] [--format json]"
---

# Review Ready

Read `~/.claude/skills/you-got-skills/skills/shared/verification-gate.md` — a PASS here requires fresh command output, not assertion.

A pre-PR gate run in 3 minutes catches the class of issues that cause review churn: debug artifacts, broken tests, MUST-level code review findings. These have the same fix cost before and after the PR opens, but a much lower interruption cost before.

## When NOT to use

- Already in the middle of `ygs-ship` — ship handles its own gates
- The branch is in a draft PR intentionally (WIP) — gate is for "ready to review" not "in progress"

## Inputs

- `--base <branch>`: branch to diff against (default: detect from `git remote show origin`)
- `--head <ref>`: ref to check (default: current HEAD)
- `--skip-ci`: skip the test stage
- `--skip-review`: skip the code review stage
- `--format json`: machine-readable output (for CI / pre-push hooks)

## Stage 1: Hygiene

Run checks from `~/.claude/skills/you-got-skills/skills/shared/hygiene-checks.md` on the merge-base diff:

```bash
BASE=$(git remote show origin 2>/dev/null | grep "HEAD branch" | sed 's/.*: //' || echo "main")
git diff $(git merge-base HEAD "origin/$BASE") HEAD
```

Grade each violation as BLOCKER or WARN per the shared file. Any BLOCKER → Stage 1 FAIL.

## Stage 2: Tests

Skip if `--skip-ci`.

Run the test suite using `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`. Show the output.

- Any test failure → Stage 2 FAIL
- No tests detected for changed paths → Stage 2 WARN (not FAIL)

## Stage 3: Quick review

Skip if `--skip-review`.

Invoke `/ygs-code-review` on the diff. Treat its findings as:
- MUST finding → Stage 3 FAIL
- SHOULD finding → Stage 3 WARN

## Output

Produce a per-stage table:

```
Stage 1 — Hygiene:   PASS | FAIL | WARN
Stage 2 — Tests:     PASS | FAIL | WARN | SKIPPED
Stage 3 — Review:    PASS | FAIL | WARN | SKIPPED

Overall: PASS | FAIL | WARN
Blockers: [list]
```

If `--format json`:
```json
{
  "status": "PASS|FAIL|WARN",
  "stages": [
    {"name": "hygiene", "status": "PASS|FAIL|WARN", "findings": [...]},
    {"name": "tests",   "status": "PASS|FAIL|WARN|SKIPPED"},
    {"name": "review",  "status": "PASS|FAIL|WARN|SKIPPED", "findings": [...]}
  ],
  "blockers": [...]
}
```

## Completion

**PASS:** Report **DONE**. Suggest: `/ygs-ship` to continue to PR creation.

**WARN:** Report **DONE_WITH_CONCERNS** — list WARN items and let user decide.

**FAIL:** Report **BLOCKED** — list every BLOCKER with file:line and fix guidance.

---

## Common Rationalizations

| Rationalization | Reality |
|----------------|---------|
| "Tests pass locally, I don't need to gate again" | The diff review catches debug artifacts and commit quality issues that test runs don't. Each stage checks a different failure mode. |
| "I'll clean it up after the PR is open" | Review feedback and cleanup changes are harder to separate once a PR is open. Gate before, not after. |
| "It's a small change, hygiene doesn't matter" | Debug artifacts and bare WIP commits are more common on small changes, not less. |
