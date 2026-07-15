# Review Scaffold

Common protocol for all review skills. Individual skills define WHAT to check — this defines HOW to run a review.

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — you own the review quality.

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

## Approach-level review

Before reporting individual findings, assess whether the overall approach is sound:
- Is this the right solution to the problem, or a well-executed wrong approach?
- Does the change address the root cause, or paper over a symptom?
- Would a simpler design achieve the same goal?

If the approach is fundamentally wrong, say so as the first finding — individual code issues are irrelevant if the direction is bad.

## Verification of findings

Before reporting any finding, verify it against the actual code:
- Confirm the code actually does what you think it does (read surrounding context)
- Test your mental model against edge cases — would the code really fail this way?
- If uncertain, mark confidence as LOW and explain what you couldn't verify

Never report a finding you haven't verified against the actual source.

## Verdict

One of:
- **Approve** — Ship it (no MUST-level findings)
- **Request changes** — Fix MUST items before merge
- **Comment** — Suggestions only, no blockers

## Completion

Report **DONE** or **DONE_WITH_CONCERNS**.
