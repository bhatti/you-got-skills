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

### 2e. Design Principles Analysis

Read `~/.claude/skills/you-got-skills/skills/shared/design-principles.md`.

Apply to the diff:

1. **SOLID** — check each principle against every class/module touched:
   - SRP: does any changed module now have >1 distinct reason to change?
   - OCP: does the new behaviour require editing existing classes? (new `if type==X` chain?)
   - LSP: does any subclass weaken a precondition or strengthen a postcondition?
   - DIP: does any high-level module now import a concrete low-level type?
   - ISP: does any interface force unrelated implementors to stub methods?

2. **Stability estimate** — for each changed module:
   - Estimate Ce (outbound imports) and Ca (inbound imports) using grep
   - Compute I = Ce / (Ca + Ce)
   - Flag if I > 0.7 for a shared/core module or if stable module is fully concrete (SAP)

3. **Hotspot check** — is the changed file a top-churn file?
   ```bash
   git log --oneline --name-only -50 -- <changed_file> 2>/dev/null | grep -v "^[0-9a-f]" | wc -l
   ```
   If > 20 touches AND I > 0.5 → flag hotspot coupling.

4. **Readability score** — for each changed function, score 5 sub-dimensions (0–2 each):
   scannability, locality of information, abstraction consistency, progressive disclosure, predictability.
   Sum / 10 = readability score. Report average across changed functions.

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

## Design Quality Metrics

Read `~/.claude/skills/you-got-skills/skills/shared/design-metrics-table.md` and emit the
canonical 8-row table here. Fill in values from Step 2e analysis.

| Principle | Score | Benchmark | Signal | Notes |
|-----------|-------|-----------|--------|-------|
| SRP violations | N | 0 ideal | 🟢/🟡/🔴 | file:line or "none found" |
| DIP violations | N | 0 ideal | 🟢/🟡/🔴 | concrete dep in domain, or "none found" |
| Cyclic deps (ADP) | N | 0 | 🟢/🔴 | cycle pair or "none detected" |
| Layer violations | N | 0 ideal | 🟢/🟡/🔴 | wrong-direction import or "none" |
| Avg instability (I) | 0.x | < 0.5 (core) | 🟢/🟡/🔴 | module name or "N/A" |
| Hotspot coupling | N files | 0 ideal | 🟢/🟡/🔴 | file name or "none" |
| Readability score | N/10 | > 7 | 🟢/🟡/🔴 | weakest sub-dimension or "all good" |
| ISP / fat interface | N | 0 ideal | 🟢/🟡/🔴 | interface name or "none found" |

## Verdict
APPROVE | REQUEST_CHANGES | COMMENT — <1 sentence rationale>
```

**Then write `findings.json`**:

```json
{
  "pr_url": "<pr_url_or_number>",
  "verdict": "APPROVE | REQUEST_CHANGES | COMMENT",
  "design_metrics": {
    "srp_violations": 0,
    "dip_violations": 0,
    "adp_cycles": 0,
    "layer_violations": 0,
    "avg_instability": 0.0,
    "hotspot_coupling": false,
    "readability_score": 0,
    "isp_violations": 0
  },
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
