---
name: ygs-worktree
description: Git worktree management — isolate feature work in a linked worktree before implementing. Use when starting work that should not affect the main workspace, or before executing multi-step implementation plans.
argument-hint: "[branch-name or feature-name]"
---

# Worktree

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.

A linked worktree gives you an isolated checkout of the repo on a new branch — your main workspace stays clean, test runs don't interfere with each other, and you can switch context instantly without stash/unstash.

## Step 1: Check if already in a worktree

```bash
git rev-parse --git-dir
```

- Returns `.git` → you're in the main worktree. Proceed to Step 2.
- Returns a path like `/path/to/repo/.git/worktrees/feature-x` → already in a linked worktree. Proceed to Step 4.
- Returns an error → not a git repo. Surface this to the user before proceeding.

Also check for submodules — if `.gitmodules` exists, note it. Worktrees share submodule state.

## Step 2: Create the worktree

**Prefer native harness tools first.** In Claude Code, use the `EnterWorktree` tool if available — it handles the setup and returns the worktree path automatically.

**Fallback — manual creation:**

```bash
# Ensure .worktrees/ is gitignored
grep -q '\.worktrees' .gitignore || echo '.worktrees/' >> .gitignore

# Create the worktree on a new branch
git worktree add .worktrees/<branch-name> -b <branch-name>
```

The branch name should match the feature: `feat/auth-refresh`, `fix/null-pointer-user-login`, etc.

## Step 3: Project setup

In the new worktree directory, install dependencies. Auto-detect:

```bash
# Node
[ -f package.json ] && npm install

# Rust
[ -f Cargo.toml ] && cargo build

# Python (pip)
[ -f requirements.txt ] && pip install -r requirements.txt

# Python (poetry)
[ -f pyproject.toml ] && poetry install

# Go
[ -f go.mod ] && go build ./...
```

Skip setup if the project shares a build cache (e.g. Cargo with shared `target-dir`).

## Step 4: Verify baseline

Before making any changes, verify the test suite passes in this worktree.

Use `~/.claude/skills/you-got-skills/skills/shared/test-runner.md` to run the suite.

**If tests fail before any changes: STOP.** Surface the failures to the user. Do not implement on a broken baseline — you won't be able to tell what you broke.

## Cleanup

When implementation is done:

1. Run `/ygs-ship` to create the PR and merge
2. After the PR is merged, prune the worktree:

```bash
git worktree remove .worktrees/<branch-name>
git worktree prune
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git worktree list` | Show all linked worktrees and their branches |
| `git worktree add .worktrees/<name> -b <branch>` | Create linked worktree on new branch |
| `git worktree add .worktrees/<name> <existing-branch>` | Check out existing branch in new worktree |
| `git worktree remove .worktrees/<name>` | Remove a worktree (must have no uncommitted changes) |
| `git worktree prune` | Clean up stale worktree metadata |
| `git rev-parse --git-dir` | Check which worktree you're currently in |

---

References:
- `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`
- `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`
