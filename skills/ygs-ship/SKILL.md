---
name: ygs-ship
description: Ship workflow — run tests, exercise the feature, review diff, bump version, update changelog, create PR. Proves it works before shipping.
---

# Ship

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — shipping means it works, not that tests pass.

## Pre-flight: Verification gate

Read `~/.claude/skills/you-got-skills/skills/shared/verification-gate.md`.

All claims in this skill — "tests pass", "feature works", "no regressions" — require fresh command output in this response.

## Step 0: Worktree check

```bash
git rev-parse --git-dir
```

If output contains `worktrees`: after PR merges, run `/ygs-worktree` cleanup.

## Step 1: Verify clean state

```bash
git status --porcelain
git log --oneline -5
```

If working tree is dirty, ask user to commit or stash first.

## Step 2: Check deploy freeze

```bash
[ -f .deploy-freeze ] && cat .deploy-freeze
```

If `.deploy-freeze` exists: report **BLOCKED** with freeze reason.

## Step 3: Detect base branch

```bash
git remote show origin | grep "HEAD branch" | sed 's/.*: //'
```

## Step 4: Gate — hygiene, tests, review

Invoke `/ygs-review-ready --base <base>` (detected in Step 3).

- **FAIL** → report **BLOCKED**. Fix before proceeding.
- **WARN** → continue. Add WARN items to the PR description under "Known minor issues."
- **PASS** → proceed.

After passing, verify against `~/.claude/skills/you-got-skills/skills/shared/definition-of-done.md`: security implications reviewed, observability in place for new critical paths, rollback path exists.

## Step 5: Exercise the feature

Tests passing is necessary but not sufficient. Verify actual behavior:
- Server: start it, hit the endpoint, confirm the response
- CLI: run with representative input
- Library: run the example or a smoke test
- UI: start the dev server and check in a browser

If you can't exercise it, state explicitly what you couldn't verify and why.

## Step 6: Diff sanity

```bash
git diff origin/<base>...HEAD --stat
```

Any files that shouldn't be committed? (env files, build artifacts, large binaries) — hygiene is already checked in Step 4, this is a quick unexpected-file scan only.

## Step 7: Version bump (if applicable)

- Patch: bug fixes
- Minor: new features (backwards compatible)
- Major: breaking changes

Ask user if unclear.

## Step 8: Update changelog (if applicable)

If CHANGELOG.md exists, add an entry under the new version.

## Step 9: Create PR

```bash
gh pr create --title "<concise title>" --body "<summary of changes>"
```

Include WARN items from Step 4 in the PR body if any.

## Step 10: Completion

Report **DONE** with tests status, version bumped (if any), PR URL. Or **BLOCKED** if Step 4 failed.

---

## Common Rationalizations

| Rationalization | Reality |
|----------------|---------|
| "Tests pass, so it works" | Tests verify what was tested. Exercise the actual feature (Step 5) to find what tests missed. |
| "It's a small change, no need for the full workflow" | Small changes break prod. The workflow exists precisely for "it's just a small change" situations. |
| "I'll add the changelog entry later" | "Later" is never. Write it now while the impact is fresh. |
| "I'll skip the diff review, I know what's in there" | Env files and build artifacts have shipped this way. Always review. |
| "The freeze check isn't relevant for my change" | Deploy freezes exist because the system is fragile right now. Every change is relevant. |
