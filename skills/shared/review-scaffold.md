# Review Scaffold

Common protocol for all review skills. Individual skills define WHAT to check — this defines HOW to run a review.

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
