---
name: ygs-git
description: Git commit discipline and release versioning — atomic commits, save-point pattern, semantic versioning, changelog hygiene. Use during development to maintain a clean, recoverable history.
argument-hint: "[optional: review current branch / release <version>]"
---

# Git Workflow

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — a clean git history is part of the deliverable.

## When NOT to use

- Release and PR creation — use `/ygs-ship` which handles version bumps, changelog, and PR creation
- Generating a formatted changelog — use `/ygs-changelog`

## Core principles

**Trunk-based development:** Work in short-lived branches (merge within 1-3 days). Long-lived branches accumulate conflict debt. Feature flags gate incomplete features — not long-lived branches.

**Commit early, commit often:** Each successful increment gets a commit. A commit is a save point — if the next step breaks something, `git reset --hard HEAD` returns to the last known-good state instantly. Do not wait for "done" to commit.

## Atomic commits

One commit = one logical thing. Not one file, not one hour, not one task — one logical change.

**~100 lines is the target size.** When a diff grows beyond ~300 lines, it's time to split.

What breaks atomicity:
- Mixing formatting changes with behavior changes — separate commits
- Mixing refactors with feature additions — separate commits
- Bundling "while I was here" cleanups — separate commits (or skip them)

Exception: causally coupled changes belong together (create migration + update model + update handlers for the same entity is one commit, not three).

## The Save-Point Pattern

The most important practice in this skill: commit after each verified slice.

```
implement slice → test → verify passes → COMMIT → next slice
```

Benefits:
- `git reset --hard HEAD` undoes a broken attempt without losing prior work
- `git bisect` becomes usable when commits are small and atomic
- Code review is easier when each commit tells one coherent story
- The final PR review diff is composed of well-named story beats

**Never commit broken code.** The invariant: `HEAD` is always in a passing state.

## Commit messages

Structure: `type(scope): what changed — why`

```
feat(auth): add OAuth2 PKCE flow — replaces implicit flow deprecated in RFC 8252
fix(queue): drain in-flight messages before shutdown — prevents message loss on SIGTERM
refactor(user): extract email validation to shared validator — same logic was duplicated in 3 places
```

**Types:** `feat` / `fix` / `refactor` / `test` / `docs` / `chore` / `perf`

**The why beats the what:** The code shows what changed. The commit message explains why.

Not: `fix bug in login` → Yes: `fix(auth): handle expired tokens gracefully — previously returned 500 instead of 401`

## Change summary (for reviews and handoffs)

When describing a change to a reviewer or in a PR, include all three parts:

```
CHANGES MADE:
- [what you changed and why]

THINGS I DIDN'T TOUCH (intentionally):
- [what's in the area but was left alone]

POTENTIAL CONCERNS:
- [anything reviewers should probe]
```

The "didn't touch" section is the most underrated part — it shows scope discipline and prevents reviewers from wondering "did they miss this?"

## Pre-commit hygiene

Before every commit:

```bash
git diff --staged          # Review exactly what's going in
```

Check for:
- Debug artifacts (`console.log`, `dbg!`, `println!`, `TODO: remove`)
- Secrets or credentials — scan before staging if the diff touches config files
- Build artifacts or large binaries accidentally staged
- Files from unrelated changes accidentally included

Then verify tests pass before committing: `git stash --include-untracked; make test; git stash pop` or equivalent.

## Semantic versioning

| Change type | Version bump | Consumer impact |
|-------------|-------------|----------------|
| **PATCH** (x.y.**Z**) | Bug fix, backwards-compatible | Safe to upgrade; no interface changes |
| **MINOR** (x.**Y**.0) | New feature, backwards-compatible | Safe to upgrade; new opt-in capabilities |
| **MAJOR** (**X**.0.0) | Breaking change | Requires consumer migration |

**Tags are the source of truth** for versions — not branch names, not filenames. A version that isn't tagged doesn't exist.

## Changelog discipline

Write the changelog entry **when you make the change**, not at release time. At release time you've forgotten the context.

Format per entry: `[type] [what changed] — [why it matters to a consumer]`

```
## [1.4.0] — 2026-08-27
### Added
- OAuth2 PKCE flow — replaces implicit grant flow; required for SPAs per RFC 8252
### Fixed
- Queue drain on shutdown — prevents message loss during rolling deployments
```

The changelog is written **for consumers**, not for the commit log. Explain the impact, not the implementation.

---

## Common Rationalizations

| Rationalization | Reality |
|----------------|---------|
| "I'll commit everything at the end when it's done" | "Done" is a long way from the last save point. One broken step loses all the work. |
| "This commit is a bit big but it's all related" | "All related" is not the same as "one logical change." Large commits make bisect useless and reviews hard. |
| "I'll write the commit message later" | The context for why you made this change is freshest right now. Write it now. |
| "The commit message doesn't need to explain why" | The code shows what changed. Without the why, the next reader has no way to evaluate whether the change is still correct as the codebase evolves. |
| "I'll update the changelog at release time" | At release time you'll write "various bug fixes." Write it now with the actual context. |

---

## Completion

Report **DONE** with:
- Commit conventions confirmed: atomic commits, descriptive messages with why, pre-commit hygiene applied
- Save-point discipline active: each verified slice has a commit
- Changelog entries written at change time (if applicable)

Suggest: `/ygs-ship` to version, create PR, and push. `/ygs-changelog` to generate a formatted release summary.
