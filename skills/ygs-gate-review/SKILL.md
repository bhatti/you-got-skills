---
name: ygs-gate-review
description: "Deep single-PR gate review: problem/solution analysis, code quality, risks, blast radius findings. Read-only — no PR mutations. Alias: @bot scope <pr>."
argument-hint: "<pr-url-or-number>"
---

# Gate Review — Deep Single-PR Analysis

Read `~/.claude/skills/you-got-skills/skills/shared/review-scaffold.md`.
Read `~/.claude/skills/you-got-skills/skills/shared/quality-checklist.md`.
Read `~/.claude/skills/you-got-skills/skills/shared/merge-queue-metrics.md`.

**READ-ONLY**: never comments on, labels, or mutates the PR. Pure gate analysis.

**Aliases**: `@bot gate-review <pr>` and `@bot scope <pr>` invoke this same workflow.

---

## Step 1: Fetch PR diff and metadata

For **GitHub** PRs, run:
```bash
gh pr view $PR_NUMBER --repo "$GH_ORG/$GH_REPO" --json title,body,author,createdAt,changedFiles,additions,deletions,labels,baseRefName
gh pr diff $PR_NUMBER --repo "$GH_ORG/$GH_REPO"
```

For **Bitbucket** PRs, run:
```bash
curl -sf "https://api.bitbucket.org/2.0/repositories/$BITBUCKET_WORKSPACE/$BITBUCKET_REPO/pullrequests/$PR_NUMBER" \
     -u "$BITBUCKET_USERNAME:$BITBUCKET_TOKEN"
curl -sf "https://api.bitbucket.org/2.0/repositories/$BITBUCKET_WORKSPACE/$BITBUCKET_REPO/pullrequests/$PR_NUMBER/diff" \
     -u "$BITBUCKET_USERNAME:$BITBUCKET_TOKEN"
```

If the repo is already cloned at `$CODEBASE_DIR`, also run `git log --oneline -20` to understand recent context.

---

## Step 2: Deep Analysis

Perform all four analysis dimensions. Every claim must cite specific file:line evidence from the diff.

### 2a. Problem / Solution

- **What problem does this PR solve?** State it in one sentence based on the PR title + description.
- **Is the solution complete?** Does it handle the problem fully, or are there obvious gaps?
- **Are there simpler alternatives?** Would a smaller change achieve the same goal?
- **Spec alignment**: Does the implementation match what the PR description promises? Note any discrepancies.

### 2b. Code Quality

Apply `shared/quality-checklist.md` at **deep** depth:
- **Correctness**: logic errors, off-by-one, null handling, wrong operator precedence
- **Complexity**: functions with CC > 10 — split them; CC > 15 — MUST flag
- **Duplication**: does this duplicate existing code? Reference the existing implementation
- **Naming**: intention-revealing? Consistent with surrounding codebase?
- **Error handling**: errors propagated with context? No silent swallowing
- **Testing**: are the changes covered by tests? New edge cases exercised?
- **Sloppiness signals**: dead code, unused imports, TODOs, debug logs left in

### 2c. Risk Assessment

Apply the blast radius and risk model from `shared/merge-queue-metrics.md`:
- **Security / auth paths**: any auth, credential, token, or session code touched?
- **Data integrity**: migrations, schema changes, or destructive operations?
- **Partial failure modes**: if this change fails mid-way, what state is the system in?
- **Race conditions**: check-then-act patterns, concurrent access, TOCTOU?
- **API surface changes**: any public API, endpoint, or contract changed?
- **SRE concerns**: timeout values, retry logic, connection pooling, resource limits?

### 2d. Category-specific deep dive

Based on the primary file category (from file paths):
- **security/authn_authz**: verify no credential leaks, proper input validation, secure defaults
- **data/migrations**: verify idempotency, backward compatibility, rollback plan
- **api**: verify schema validation, error codes, backward compatibility
- **sre/infra**: verify resource limits, health checks, rollback procedure

---

## Step 3: Verdict and Output

**Write analysis summary to stdout** before creating `findings.json`. Output:

```
## PR Purpose
<2-3 sentences: what problem, what approach, expected outcome>

## Implementation Quality
<concrete observations — not "looks good". What specifically is well done or problematic?>

## Key Risks
<risks by category with diff evidence. "No risks" is only valid if you explicitly checked each category above>

## Verdict
APPROVE | REQUEST_CHANGES | COMMENT — <1 sentence rationale>
```

**Then write `findings.json`**:

```json
{
  "pr_url": "<pr_url_or_number>",
  "verdict": "APPROVE | REQUEST_CHANGES | COMMENT",
  "findings": [
    {
      "severity": "CRITICAL | HIGH | MEDIUM | LOW",
      "confidence": "HIGH | MEDIUM | LOW",
      "title": "<short one-line title>",
      "file": "<path/to/file or empty>",
      "line": null,
      "domain": "correctness | security | api | sre | quality",
      "description": "<what is wrong and why it matters>",
      "fix": "<concrete suggested fix>"
    }
  ],
  "summary": "<one sentence overall assessment>"
}
```

- Use `REQUEST_CHANGES` when any CRITICAL or HIGH finding exists.
- Use `COMMENT` for MEDIUM findings only (no blockers).
- Use `APPROVE` only when no findings above LOW severity.
- An empty `findings` array with verdict `APPROVE` means you checked all dimensions and found nothing — write a non-trivial `summary` explaining what you verified.

**Output ONLY this JSON on the last line** (required by the runner):
`{"status":"DONE","findings_count":<N>,"verdict":"<verdict>","summary":"<one sentence>"}`

---

## Anti-patterns — do not fall into these

| Anti-pattern | Why it fails |
|---|---|
| "No issues found" with no evidence | Did you check each dimension? List what you verified. |
| Generic praise ("clean implementation") | Cite specific code. What specifically is clean? |
| Re-deriving blast_radius from scratch | `scope.json` has pre-computed values — use them for context, not duplication |
| Skipping Step 2c because "no auth code" | Explicitly confirm each risk category was checked |

**Shared refs:** `shared/review-scaffold.md` (severity/confidence levels, finding format), `shared/quality-checklist.md` (CC thresholds, sloppiness signals), `shared/merge-queue-metrics.md` (blast radius, risk tiers)
