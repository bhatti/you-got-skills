---
name: ygs-ship
description: Ship workflow — run tests, review diff, bump version, update changelog, create PR. Use when code is ready to merge.
---

# Ship

## Step 1: Verify clean state

```bash
git status --porcelain
git log --oneline -5
```

If working tree is dirty, ask user to commit or stash first.

## Step 2: Detect base branch

```bash
git remote show origin | grep "HEAD branch" | sed 's/.*: //'
```

## Step 3: Run tests

```bash
[ -f Makefile ] && make test
[ -f package.json ] && npm test
[ -f Cargo.toml ] && cargo test
[ -f pytest.ini ] && pytest
[ -f go.mod ] && go test ./...
```

If tests fail: **BLOCKED** — fix tests before shipping.

## Step 4: Review diff against base

```bash
git diff origin/<base>...HEAD --stat
```

Quick sanity check:
- Any files that shouldn't be committed? (env files, build artifacts, large binaries)
- Any debug code left in? (console.log, debugger, TODO hacks)
- Does the diff match what you intended to ship?

## Step 5: Version bump (if applicable)

If the project has a VERSION file, package.json version, or Cargo.toml version:
- Patch: bug fixes
- Minor: new features (backwards compatible)
- Major: breaking changes

Ask user which bump is appropriate if unclear.

## Step 6: Update changelog (if applicable)

If a CHANGELOG.md exists, add an entry under the new version with a summary of changes.

## Step 7: Create PR

If `gh` is available and changes are on a feature branch:

```bash
gh pr create --title "<concise title>" --body "<summary of changes>"
```

If not on a feature branch or `gh` unavailable, report what would be done.

## Step 8: Completion

Report **DONE** with:
- Tests status
- Version bumped (if applicable)
- PR URL (if created)

Or **BLOCKED** if tests fail or issues found.
