---
name: ygs-ship
description: Ship workflow — run tests, exercise the feature, review diff, bump version, update changelog, create PR. Proves it works before shipping.
---

# Ship

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — shipping means it works, not that tests pass.

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

If `.deploy-freeze` exists: report **BLOCKED** with freeze reason. Suggest `/ygs-sre-review --unfreeze` to lift.

## Step 3: Detect base branch

```bash
git remote show origin | grep "HEAD branch" | sed 's/.*: //'
```

## Step 4: Run tests

Use the polyglot test runner from `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.

If tests fail: **BLOCKED** — fix tests before shipping.

## Step 5: Exercise the feature

Tests passing is necessary but not sufficient. Before shipping, verify the actual behavior:
- If it's a server: start it, hit the endpoint, confirm the response
- If it's a CLI: run the command with representative input
- If it's a library: run the example or a quick smoke test
- If it's a UI change: start the dev server and check it in a browser

If you can't exercise it (no dev environment, external dependency), explicitly state what you couldn't verify. Don't ship blind.

## Step 6: Review diff against base

```bash
git diff origin/<base>...HEAD --stat
```

Quick sanity check:
- Any files that shouldn't be committed? (env files, build artifacts, large binaries)
- Any debug code left in? (console.log, debugger, TODO hacks)
- Does the diff match what you intended to ship?

## Step 7: Version bump (if applicable)

If the project has a VERSION file, package.json version, or Cargo.toml version:
- Patch: bug fixes
- Minor: new features (backwards compatible)
- Major: breaking changes

Ask user which bump is appropriate if unclear.

## Step 8: Update changelog (if applicable)

If a CHANGELOG.md exists, add an entry under the new version with a summary of changes.

## Step 9: Create PR

If `gh` is available and changes are on a feature branch:

```bash
gh pr create --title "<concise title>" --body "<summary of changes>"
```

If not on a feature branch or `gh` unavailable, report what would be done.

## Step 10: Completion

Report **DONE** with:
- Tests status
- Version bumped (if applicable)
- PR URL (if created)

Or **BLOCKED** if tests fail or issues found.
