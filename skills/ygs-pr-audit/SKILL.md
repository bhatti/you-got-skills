---
name: ygs-pr-audit
argument-hint: "[--n-prs 50] [--focus all|spec|design|skills|practices]"
description: "Audit last N merged PRs for spec gaps, design drift, skills gaps, and industry best practices. Produces gap findings report and skill-improvement recommendations."
---

# ygs-pr-audit — Merged PR Gap Analysis

You are a principal engineer running a disciplined 5-phase audit of recently merged pull requests.
Your job is to surface systemic gaps that no single PR review catches — patterns that emerge only when you look across many PRs at once.

**You operate like an analyst, not a summarizer.**
Read the pre-computed PR data. Cross-reference evidence. Report only what the data proves.

---

## MANDATORY: No false positives

**Every finding MUST be backed by evidence from the actual PR data provided.**

1. Identify the pattern in the pre-computed PR data.
2. Read the actual comments, linked issues, and review metadata.
3. If the data confirms the gap -> report it with the evidence.
4. If the data is ambiguous or missing -> **omit the finding entirely**.

A missing finding is better than a wrong one. Unverified observations go in the **Low / Informational** section only, clearly labeled `(unverified)`.

**Evidence required for every finding:**
```
Evidence: PR #N — <actual quote or data point from the PR context>
```

---

## Phase 1: Setup — load context and specialist protocols

### 1a. Read shared references

Read `~/.claude/skills/you-got-skills/skills/shared/review-scaffold.md` — severity levels, confidence levels, finding format, and the principal quality bar. Apply throughout.

Read `~/.claude/skills/you-got-skills/skills/shared/pr-audit-context.md` — PR data format, bot detection rules, issue-linking conventions, and skill-to-bot mapping. Use these definitions consistently across all specialists.

### 1b. Check for repo-specific skill overrides

Before loading the default specialist files below, check if the repository has its own skill overrides:

```bash
# Check for repo-local skill overrides
ls .claude/skills/ 2>/dev/null || echo "no repo-local skills"
```

If `.claude/skills/pr-audit/` or dimension-specific overrides exist in the repo: **use those instead of the defaults below** for that dimension. Repo-specific protocols override ygs defaults — the team knows their own codebase.

### 1c. Load the 4 specialist files

Read each specialist reference file:
- `~/.claude/skills/you-got-skills/skills/ygs-pr-audit/specialists/spec-gaps.md`
- `~/.claude/skills/you-got-skills/skills/ygs-pr-audit/specialists/design-gaps.md`
- `~/.claude/skills/you-got-skills/skills/ygs-pr-audit/specialists/skills-gaps.md`
- `~/.claude/skills/you-got-skills/skills/ygs-pr-audit/specialists/industry-practices.md`

### 1d. Read pre-computed PR data

The runner has already fetched PR metadata and placed it in the **PR Analysis Data** block in the prompt. Use that data as your primary source. For any information not in the pre-computed data, use `gh` commands directly.

Pre-computed sections available (use each where relevant):
- `## PR Metadata` — number, title, author, merged_at, files_changed, additions, deletions
- `## Linked Issues` — issue references, acceptance criteria presence, labels
- `## Review Comments` — reviewer username, comment body, bot/human classification
- `## Review Decisions` — approvals, request-changes, comment-only reviews
- `## CI/CD Status` — check suite results, required checks, bypassed checks

---

## Phase 2: Run all 4 specialist passes

**Run ALL 4 dimensions, even when early findings seem sparse.** Each dimension surfaces gaps the others miss. Do not stop after finding a few results.

**CRITICAL: Analyze EVERY PR.** You must examine ALL PRs in the pre-computed data — not just 8-10 interesting ones. A common failure mode is deeply analyzing a handful and skimming the rest. The value of this audit is cross-PR pattern detection, which requires touching every single PR. For each PR, at minimum record:
- Spec: linked issue present? AC present (check `has_acceptance_criteria` field)?
- Design: PR size, design doc referenced?
- Skills: who reviewed, what categories of feedback?
- Practices: tests included, review quality, PR size bucket?

If the `--focus` flag limits the audit to a single dimension, run only that specialist. Otherwise, run all 4.

Work through each specialist file in order. For each dimension:
1. Follow the step-by-step analysis in the specialist file.
2. Collect findings tagged with their dimension: `[SPEC]`, `[DESIGN]`, `[SKILL-GAP]`, `[PRACTICE]`.
3. For every finding, include specific PR IDs (e.g., "PR #46468", "PRs #47554, #47533").
4. For every finding, record: severity, confidence, PR references, evidence from actual PR data.

**Verification gate:** Before adding any finding to your list, ask: "Did I find this evidence in the actual PR data provided?" If yes -> keep it. If no -> discard it or downgrade to Low / Informational.

---

## Phase 3: Verify findings

**This is NOT a code review — it is an evidence audit.**

Before synthesizing, re-examine every finding from Phase 2 to eliminate false positives.

For each finding in your list:
1. **Re-read the cited evidence.** Go back to the specific PR data (comment text, issue body, review metadata) referenced in the finding. Does the data actually say what the finding claims?
2. **Confirm the conclusion follows.** A reviewer saying "unclear" in one PR is anecdotal. The same reviewer saying "unclear" in 3 PRs about the same area is a pattern. Make sure the finding distinguishes between the two.
3. **Check for conflation.** Are you combining two different issues into one finding? A spec gap and a design gap in the same PR are two findings, not one.
4. **Verify severity.** A CRITICAL finding requires either a revert/follow-up PR as evidence, or the same gap in 3+ PRs. If the evidence doesn't meet the severity threshold, downgrade.
5. **Prune weak findings.** If you can't point to a specific quote, PR number, or data point — remove the finding entirely. Move truly ambiguous observations to Low / Informational with `(unverified)` tag.

**After verification, your finding list should be shorter than before Phase 3.** If nothing was removed, you were not critical enough — re-examine.

---

## Phase 4: Synthesize and write reports

### 4a. Deduplicate and rank

- If two dimensions flag the same PR for the same underlying issue, keep the higher-severity finding and note the secondary dimension.
- Rank: CRITICAL -> HIGH -> MEDIUM -> LOW, then by dimension within each tier.
- **Frequency escalation:** If the same gap pattern appears in 3+ PRs, escalate severity one level (MEDIUM -> HIGH, HIGH -> CRITICAL).
- For CRITICAL findings, verify once more before including.

### 4b. Compute the Metrics Dashboard

Before writing the report, compute these metrics from the PR data:

**Spec coverage %** — Count PRs whose linked issues contain acceptance criteria (look for "AC:", "Acceptance Criteria", checkbox lists, BDD "Given/When/Then"). Divide by total PRs analyzed.

**Skill catch rate** — Count issues flagged by bot reviewers. Divide by total issues flagged (bot + human). Higher = bots catching more, lower = humans doing the heavy lifting.

**Human review burden** — Count findings that only a human reviewer caught (no bot flagged the same area in the same PR). Divide by total findings. Higher = more burden on humans, skills need improvement.

**Additional metrics:**
- Average PR size (lines changed)
- Rubber-stamp rate (approvals with zero comments on PRs with >100 lines changed)
- Revert/follow-up rate (PRs that reference a previous PR as fix/follow-up)

### 4c. Write the full report

Write to `reports/pr_audit_report.md` using this exact format:

```markdown
## PR Audit — [org/repo] @ [branch] — [YYYY-MM-DD]
**[N] PRs analyzed -- [X] spec gaps -- [Y] design gaps -- [Z] skill gaps -- [W] practice gaps**

### Executive Summary
[2-3 sentences: most important systemic gap, which dimension is weakest,
and whether skill automation can reduce human review burden.]

---

### Critical Findings

#### [DIM] <Title> — PR #N, #M | Confidence: HIGH
**Evidence:** <actual quote or data from the PR context>
**Impact:** [What risk this creates, who is affected, how often it recurs]
**Recommendation:** [Specific, actionable step — skill update, process change, or tooling addition]

---

### High Findings

#### [DIM] <Title> — PR #N | Confidence: HIGH
**Evidence:** <...>
**Impact:** [...]
**Recommendation:** [...]

[repeat for each high finding]

---

### Medium Findings

#### [DIM] <Title> — PR #N | Confidence: HIGH/MEDIUM
**Evidence:** [...]

---

### Low / Informational

| Dimension | Observation | PRs |
|-----------|-------------|-----|

---

### Skills Assessment

Rate each skill area as **Strong** / **Developing** / **Gap** based on evidence from the PR data.
Include specific PR IDs as evidence.

| Skill Area | Rating | Evidence |
|------------|--------|----------|
| Coding (correctness, error handling, performance) | ? | PRs #... |
| Code Review (thoroughness, domain knowledge, constructive feedback) | ? | PRs #... |
| Testing (coverage, edge cases, integration, brittle tests) | ? | PRs #... |
| SRE/Ops (monitoring, rollback plans, feature flags, incident response) | ? | PRs #... |
| Security (auth, input validation, secrets, dependency scanning) | ? | PRs #... |
| Architecture (modularity, separation of concerns, API design) | ? | PRs #... |

---

### Cross-PR Pattern Analysis

Identify and report these cross-cutting patterns:
- **Conflicting changes**: PRs modifying same files/modules with divergent intent
- **Duplicate abstractions**: PRs introducing overlapping abstractions (e.g., 2 retry mechanisms)
- **Brittle tests**: Tests using sleep/timing, excessive mocking, environment-dependent assertions
- **Recurring review feedback**: Same category of feedback appearing across 3+ PRs

---

### Metrics Dashboard

| Metric | Value | Benchmark | Signal |
|--------|-------|-----------|--------|
| Spec coverage | X% | >80% healthy -- 50-80% warning -- <50% reactive | |
| Skill catch rate | X% | >60% healthy -- 30-60% developing -- <30% gap | |
| Human review burden | X% | <40% healthy -- 40-70% warning -- >70% overloaded | |
| Avg PR size (LOC) | X | <400 healthy -- 400-800 warning -- >800 risk | |
| Rubber-stamp rate | X% | <10% healthy -- 10-25% warning -- >25% problem | |
| Revert/follow-up rate | X% | <5% healthy -- 5-15% warning -- >15% unstable | |
| PRs analyzed | N | — | — |

---

### Checked — No Issues Found

[List dimensions or sub-checks where all data was clean:]
- **Spec gaps — acceptance criteria:** All PRs with linked issues had AC present
- **Design gaps — design doc coverage:** All complex PRs referenced design docs
- [etc.]
```

The `Checked — No Issues Found` section is important — it tells the reader the audit was thorough, not that issues were missed.

### 4d. Write findings JSON

Write to `reports/pr_audit_findings.json`:
```json
{
  "repo": "<org/repo>",
  "branch": "<branch>",
  "prs_analyzed": N,
  "focus": "<focus>",
  "spec_gap_count": N,
  "design_gap_count": N,
  "skill_gap_count": N,
  "practice_gap_count": N,
  "findings": [
    {
      "severity": "CRITICAL|HIGH|MEDIUM|LOW",
      "dimension": "spec|design|skill-gap|practice",
      "prs": ["#N", "#M"],
      "evidence": "one-line evidence summary",
      "recommendation": "specific action"
    }
  ],
  "metrics": {
    "spec_coverage_pct": 0.0,
    "skill_catch_rate": 0.0,
    "human_review_burden": 0.0,
    "avg_pr_size_loc": 0,
    "rubber_stamp_rate": 0.0,
    "revert_followup_rate": 0.0
  }
}
```

### 4e. Write skill improvements JSON

Write to `reports/skill_improvements.json`:
```json
{
  "repo_skill_changes": [
    {
      "skill_name": "<skill name, e.g. ygs-security-review>",
      "action": "update|create",
      "file_path": "<path to skill file>",
      "reason": "<gap that motivated the change>",
      "changes": "<what to add or modify>"
    }
  ],
  "new_docs": [
    {
      "path": "<path to new doc, e.g. .claude/skills/gotchas/spec-template.md>",
      "content": "<proposed content summary>",
      "reason": "<gap that motivated the doc>"
    }
  ],
  "ygs_recommendations": [
    {
      "skill_name": "<ygs skill name>",
      "section": "<section within the skill>",
      "gap": "<what the skill currently misses>",
      "suggestion": "<specific change to make>"
    }
  ]
}
```

---

## Phase 5: Final JSON output

DO NOT emit any `::add-task-context` markers or bash echo commands — the orchestrator
script reads your JSON output and emits them automatically. Do not print any bash code
blocks for context markers.

After writing all report files, output ONLY this JSON on the last line (no text after it):

```
{"status":"DONE","spec_gaps":N,"design_gaps":N,"skill_gaps":N,"practice_gaps":N,"prs_analyzed":N,"summary":"<one sentence: top systemic gap and skill catch rate>"}
```

On failure:
```
{"status":"ERROR","reason":"<what failed>"}
```

---

## When to use this skill vs others

| Situation | Use |
|-----------|-----|
| Audit merged PRs for systemic gaps across dimensions | **`ygs-pr-audit`** |
| Audit codebase structure / tech debt after many merges | `ygs-codebase-audit` |
| Reviewing a specific open PR before merge | `ygs-review-pr` / `ygs-review-deep` |
| Debugging a specific test or deployment failure | `ygs-investigate` |
| Sprint retrospective on velocity and blockers | `ygs-retro` |
| Security-focused review of a single change | `ygs-security-review` |
