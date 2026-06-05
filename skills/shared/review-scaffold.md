# Review Scaffold

Common protocol for all review skills. Individual skills define WHAT to check — this defines HOW to run a review.

## Gather context

Before reviewing code, understand intent:

1. Read commit messages on the branch (`git log --oneline main..HEAD`)
2. Check for a linked task or PR description — what problem is being solved?
3. If `tasks/in-progress/*.md` exists, read the active task for acceptance criteria

Use this context to distinguish "intentional trade-off" from "mistake" during review. A change that looks wrong in isolation may be correct given the goal.

## Project review rules

If the project has `.ygs/review-rules.md`, read it and apply as additional review criteria. This file contains project-specific conventions (e.g., "all API handlers must validate tenant context", "no direct DB calls outside repository layer").

Findings from project rules use the same severity/confidence format as all other findings.

## Getting the diff

```bash
git diff main...HEAD --stat 2>/dev/null || git diff master...HEAD --stat 2>/dev/null
```

If a PR number is given and `gh` is available:

```bash
gh pr diff <number>
```

Read full file contents for changed files (not just diff hunks).

## Severity classification

Use RFC 2119 keywords consistently across all reviews:

| Severity | Meaning | Merge impact |
|----------|---------|-------------|
| **MUST** | Violating this blocks approval | Request changes |
| **SHOULD** | Strong recommendation with clear rationale | Comment |
| **MAY** | Suggestion, take or leave | Comment |

## Confidence levels

| Level | Meaning |
|-------|---------|
| **HIGH** | Certain — mechanical check, provable |
| **MEDIUM** | Likely — judgment call with evidence |
| **LOW** | Preference — reasonable people may disagree |

## Finding format

For each finding:
- **Severity:** MUST | SHOULD | MAY
- **Confidence:** HIGH | MEDIUM | LOW
- **File:line** reference
- **Issue** — what's wrong
- **Fix** — suggested resolution

## Verdict

One of:
- **Approve** — Ship it (no MUST-level findings)
- **Request changes** — Fix MUST items before merge
- **Comment** — Suggestions only, no blockers

## Completion

Report **DONE** or **DONE_WITH_CONCERNS**.
