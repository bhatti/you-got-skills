---
name: ygs-pr-audit
argument-hint: "[--n-prs 50] [--focus all|spec|design|skills|practices]"
description: "Audit PRs for spec gaps, design drift, skills gaps, and industry best practices. Produces gap findings report and skill-improvement recommendations."
---

# ygs-pr-audit — PR Gap Analysis

You are a principal engineer running a disciplined 5-phase audit of pull requests.
Your job is to surface systemic gaps that no single PR review catches — patterns that emerge only when you look across many PRs at once.

## Scope

The PR data block in the prompt includes a `state` field for each PR: `open`, `merged`, or `closed`.

- When a PR's `state` is `open`: it has not yet been merged. Use "is open" (not "was merged") in all findings. Example: "PR #47862 is open with 0 substantive human review to date."
- When a PR's `state` is `merged`: use "was merged" in findings.
- When a PR's `state` is `declined`: the PR was REJECTED — do NOT generate "merged without review" findings. See the Phase 2 state gate for full rules.
- When a PR's `state` is `closed`: treat as declined (closed without merging). Note positively if there was review engagement; otherwise note as a process observation only.

If `PR_AUDIT_TEAM_MEMBERS` env var is set, the analysis is scoped to PRs authored or reviewed by those contributors (comma-separated display names / GitHub logins). Note the team scope in the executive summary.

If `JIRA_SPACE` env var is set, the analysis is scoped to PRs referencing Jira issues tagged with that team name (via the `Eng Scrum Team` custom field). For pure-GitHub jobs, `JIRA_SPACE` maps to a GitHub label filter instead. Note the team scope in the executive summary.

If `JIRA_BOARDS` env var is set, the analysis is scoped to PRs referencing issues on those Jira board IDs (all board issues, not sprint-scoped). Note the board scope in the executive summary.

If `PR_AUDIT_FILTER` env var is set, it is either `label=X` (GitHub label filter) or `field=value` (Jira JQL custom-field filter). Note the filter scope in the executive summary.

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

Read `~/.claude/skills/you-got-skills/skills/shared/merge-queue-metrics.md` — blast radius, scope classification, risk dimension definitions. Use for the design-gaps Step 8 blast radius assessment and Metrics Dashboard blast radius rows.

### 1b. Check for repo-specific skill overrides

Follow the **Repo-local skill consolidation** protocol from `shared/review-scaffold.md`.

```bash
# Check for repo-local skill overrides
ls .claude/skills/ 2>/dev/null \
  | grep -E "pr-audit|spec|design|practices" \
  || echo "no repo-local overrides"
```

For each dimension where a repo-local file exists:
- Read the repo-local file **first** as primary (project-specific rules — priority on conflicts)
- **Also** read the ygs specialist below (the ygs paths below always point to the ygs baseline regardless of any override)
- Consolidate: repo wins on conflicts; ygs fills any gap not covered; note override in Informational

Never drop ygs checks silently — either apply them or note "repo overrides this check."

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

**New pre-computed review-depth fields** (use directly — do not re-compute from comment text):
- `Substantive human comments` — count of human comments that are NOT rubber-stamp phrases
- `Rubber-stamp approvers` — list of approvers who left zero substantive comments on this PR
- `Author type: AI/bot-authored PR` — present when the PR author's username ends with `bot` or matches known AI coding agents (Copilot, Claude agent, etc.)

---

## Phase 2: Run all 4 specialist passes

**Run ALL 4 dimensions, even when early findings seem sparse.** Each dimension surfaces gaps the others miss. Do not stop after finding a few results.

**CRITICAL: Analyze EVERY PR.** You must examine ALL PRs in the pre-computed data — not just 8-10 interesting ones. A common failure mode is deeply analyzing a handful and skimming the rest. The value of this audit is cross-PR pattern detection, which requires touching every single PR. For each PR, at minimum record:
- Spec: linked issue present? AC present (check `has_acceptance_criteria` field)?
- Design: PR size, design doc referenced?
- Skills: who reviewed, what categories of feedback?
- Practices: tests included, review quality, PR size bucket?

If the `--focus` flag limits the audit to a single dimension, run only that specialist. Otherwise, run all 4.

### ⚠️ MANDATORY: PR State Gate — check before writing any finding

Before recording any finding, inspect the PR's `state` field:

| state | Meaning | Finding rule |
|-------|---------|--------------|
| `merged` | PR landed in the codebase | Generate findings normally — the risk has materialized |
| `open` | PR is currently in review | Frame as a current risk ("is open with…"), not a historical failure |
| `declined` | PR was rejected — review process worked | **SKIP** "merged without review" and "scope explosion that landed" findings. Note high-engagement declines as **positive evidence** the process caught the issue. Only flag a declined PR when it had zero review activity AND zero comments before decline. |

**Do NOT generate a CRITICAL or HIGH finding about a declined PR** unless it had zero human engagement of any kind (no comments, no approvals, no requests-for-changes) before being declined. A bot-authored PR that was closed without merging is not a sign of process failure — it is the process working.

Work through each specialist file in order. For each dimension:
1. Follow the step-by-step analysis in the specialist file.
2. Collect findings tagged with their dimension: `[SPEC]`, `[DESIGN]`, `[SKILL-GAP]`, `[PRACTICE]`.
3. For every finding, include specific PR IDs (e.g., "PR #46468", "PRs #47554, #47533").
4. For every finding, record: severity, confidence, PR references, evidence from actual PR data.
5. For every finding that references a PR, use state-accurate language: "is open" for open PRs, "was merged" for merged PRs.

**Verification gate:** Before adding any finding to your list, ask: "Did I find this evidence in the actual PR data provided?" If yes -> keep it. If no -> discard it or downgrade to Low / Informational.

---

## Phase 2.5: Targeted Deep Review (conditional)

After Phase 2, identify PRs that qualify for specialized deep review:

**Security-sensitive PRs**: PR file paths contain any of: `auth`, `iam`, `credential`, `permission`, `secret`, `sandbox`, `rbac`, `encrypt`, `token`, `oauth`, `saml`, `acl`. OR human reviewers used security keywords ("injection", "auth bypass", "privilege", "credential", "vulnerability").

**Large under-reviewed PRs**: PR >800 LOC with fewer than 3 human review comments.

For qualifying PRs (limit: `MAX_DEEP_REVIEWS` env var, default 2 total), invoke the appropriate skill:

```
For security-sensitive PRs:   use the Skill tool → /ygs-security-review
For large under-reviewed PRs: use the Skill tool → /ygs-review-deep
```

Tag findings from deep reviews with `(deep-review)` and include them in the appropriate finding sections.

**Skip Phase 2.5 if:**
- `--focus` is not `all` or `skills`
- `MAX_DEEP_REVIEWS` env var is `0`
- No qualifying PRs found
- The PR diff is not available in the working directory (pr-audit analyzes history, not open PRs)

**Note**: Phase 2.5 is best-effort. If the deep-review skill is not installed or fails, continue to Phase 3 with Phase 2 findings only.

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

Before writing the report, compute these metrics from the PR data.

**MANDATORY — Fixed row set, fixed order:** The dashboard table MUST contain exactly the rows defined in the template below, in the order shown. Never add, remove, or rename rows. If a value cannot be computed (e.g., no bot-authored PRs), write `N/A` in the Value column. Extra observations belong in the findings sections, not as extra dashboard rows. Two audit runs on the same repo must produce dashboards that differ only in values, not in row structure.

**Spec coverage %** — Use the pre-computed `has_acceptance_criteria` field from the PR data. For entries marked `false`, read the `Issue description excerpt` and apply semantic judgment — prose that clearly describes the desired fix or behavior counts as requirements (see spec-gaps specialist). Exclude access-denied and no-issue PRs from the denominator. Formula: (Has AC) / (Has AC + No AC confirmed).

**CI catch rate** — Count issues flagged by CI bots (build/test/lint failures). Divide by total issues flagged (all bots + human). This measures pipeline health, not code-review skill quality.

**Code-review skill catch rate** — Count issues flagged by code-review bots (Claude PR Review Agent, CodeRabbit, etc.). Divide by total issues flagged by code-review bots + humans. Higher = review skills effective. NEVER mix CI bots into this metric.

**Bot-finding follow-through rate** — Count code-review bot findings that were resolved or acknowledged before merge. Divide by total code-review bot findings. Below 70% = process gap (bot findings being ignored).

**Human review burden** — Count findings that only a human reviewer caught (no code-review bot flagged the same area in the same PR). Divide by total findings. Higher = more burden on humans, skills need improvement.

**Additional metrics — include ALL of these in the dashboard table:**

- **PR State Breakdown** — merged=N  open=N  declined=N  total=N. Use the pre-computed `pr_state_summary` from the prompt header. Declined PRs excluded from all gap-rate denominators.
- **PR Size Distribution** — count PRs in each bucket using the `size_bucket` field (xs/s/m/l/xl). Add a median_loc value. Formula: use `additions + deletions` per PR.
- **Average Changed Files/PR** — avg `files_changed` across merged PRs.
- **Large PR (xl) Review Coverage** — % of xl-bucket PRs (≥1000 LOC) that had ≥1 substantive human review comment. Benchmark: 100%. Below 80% = HIGH gap.
- **Large PR Human Comments Avg** — average human comment count on PRs >400 LOC vs PRs ≤400 LOC. A lower ratio for large PRs signals review burden scaling problem.
- **Review Depth vs Size Correlation** — does review quality (human comments, approvers) increase with PR size? State "Yes — larger PRs get more review" or "No — review depth is flat regardless of size (risk)".
- Average PR size (lines changed)
- Large PR review depth: average human review comments on PRs >400 LOC
- Security review invocation rate: % of security-sensitive PRs (touching auth/IAM/credentials) that had a security skill invoked (check `.claude/skills/` for any security-review skill in the repo)
- Rubber-stamp rate: use `Rubber-stamp approvers` field — (high-blast-radius PRs where ALL approvers are rubber-stamp) / (all high-blast-radius PRs). Severity by blast-radius: auth/billing/query-engine = HIGH; config/API surface = MEDIUM; other = LOW.
  **NEVER conflate with "no review"**: rubber-stamp = approved but shallow; no review = not approved at all.
  Rubber-stamp phrases: LGTM, +1, looks good, ship it, emoji-only, silent approval (no comment), "approved", "no issues".
- Bot-authored PR review depth: (bot-authored PRs with ≥1 substantive human comment) / (all bot-authored PRs). Benchmark: 100% — every bot-authored PR should have at least one reviewer who documented what they validated.
- Revert/follow-up rate (PRs that reference a previous PR as fix/follow-up)

**Security review invocation rate — recommendation guidance:**
When this rate is low, do NOT tell teams to invoke `/ygs-security-review` (it may not exist in their repo).
Instead recommend the repo **add or improve a security review skill** in `.claude/skills/` (e.g., create `.claude/skills/security-review/SKILL.md`). You may mention `/ygs-security-review` as a reference example they can copy from.

### 4c. Write the full report and Slack digest

Write **two** output files:

**1. `reports/slack_summary.md`** — Slack-optimized digest.
This is what appears in Slack by default (users pass `--full` to get the complete report).
Include ALL critical and high findings; keep each bullet to one line (title + PRs + severity).

```markdown
### Executive Summary
[2–3 sentences: overall verdict, most critical systemic gap, whether skill automation can reduce review burden]

### Critical & High Findings
• [DIM] **Title** — PRs #N, #M | CRITICAL
• [DIM] **Title** — PR #N | HIGH
• [DIM] **Title** — PRs #N, #M, #P | HIGH
[list every critical and high finding, one line each; omit medium/low]

### Skills Assessment
*Spec* ⚠️ Gap · *Design* ✅ Strong · *Skills* ⚠️ Developing · *Practices* ❌ Gap
[one line covering all four audit dimensions]

### Metrics & Next Steps
*N PRs analyzed · X spec · Y design · Z skill · W practice gaps*
**Top priority:** [one concrete action item]
**Positive pattern:** [one strength worth calling out]
```

Use ✅ Strong / ⚠️ Developing / ❌ Gap for each dimension.
One line per finding — no Evidence/Impact sub-bullets. The full report has the detail.

**2. `reports/pr_audit_report.md`** — the complete report. Format:

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

### Positive Patterns

Highlight exemplary behavior from this PR set. Recognition reinforces good practices and shows
the audit is balanced, not just a defect hunt.

- **Exceptional reviewers**: Name reviewers who demonstrated deep domain expertise or caught
  high-impact issues before merge (e.g., "Abbas Mashayekh caught an auth bypass in PR #xxx;
  Zeynep Acar identified 4 architectural issues across PRs #yyy, #zzz, #www, #vvv.")
- **High-quality PRs**: PRs with excellent descriptions, comprehensive tests, or well-structured
  rollback plans
- **Good practices in action**: Teams or individuals who consistently provide constructive,
  detailed feedback that prevents rework or production incidents

If no standout positive patterns are found, write: "No exceptional patterns identified in this batch."

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

### Recommended Skill Updates

List every skill that should be updated or created, with the specific file path, what to change, and which PRs motivated it. This section drives the `skill_improvements.json` output and tells the reader exactly what to fix.

| Action | Skill / File Path | What to Add/Change | Motivated by PRs |
|--------|-------------------|--------------------|-----------------|
| Update | `.claude/skills/security-review/SKILL.md` | Add SQL injection detection for f-string patterns | #45, #67 |
| Create | `.claude/skills/gotchas/pagination.md` | Document pagination edge cases (off-by-one, empty pages) | #34, #56, #78 |
| Update | `ygs-review-pr` (upstream) | Add check for missing rollback plans in high-risk PRs | #12, #89 |

If no skill updates are needed, write: "No skill updates recommended — current skill coverage is adequate."

---

### Cross-PR Pattern Analysis

Identify and report these cross-cutting patterns:
- **Conflicting changes**: PRs modifying same files/modules with divergent intent
- **Duplicate abstractions**: PRs introducing overlapping abstractions (e.g., 2 retry mechanisms)
- **Brittle tests**: Tests using sleep/timing, excessive mocking, environment-dependent assertions
- **Recurring review feedback**: Same category of feedback appearing across 3+ PRs

---

### Metrics Dashboard

| Metric | Value | Benchmark | Signal | Description |
|--------|-------|-----------|--------|-------------|
| Spec coverage | X% | >80% healthy / 50-80% warning / <50% reactive | | % of PRs that have a linked issue with acceptance criteria. No spec = no way to verify the right thing was built. |
| CI catch rate | X% | pipeline health — not review skill quality | | % of all flagged issues that were caught by CI bots (build/test/lint). Measures pipeline health; do NOT mix with code-review skill quality. |
| Code-review skill catch rate | X% | >60% healthy / 30-60% developing / <30% gap | | % of issues caught by AI code-review tools (vs. human + tool total). Measures how effectively automated skills substitute for manual review. |
| Bot-finding follow-through | X% | >90% healthy / 70-90% warning / <70% process gap | | % of code-review bot findings that were resolved or acknowledged before merge. Below 70% = reviewers are rubber-stamping over bot findings. |
| Human review burden | X% | <40% healthy / 40-70% warning / >70% overloaded | | % of findings only humans caught (no bot caught the same area in the same PR). High = over-reliance on human reviewers; skills need improvement. |
| Avg PR size (LOC) | X | <400 healthy / 400-800 warning / >800 risk | | Mean lines changed per PR. Large PRs receive shallower reviews and carry higher revert blast radius. |
| Large PR review depth (avg comments, PRs >400 LOC) | X | >5 healthy / 2-5 warning / <2 gap | | Average substantive human review comments on PRs over 400 LOC. Low = reviewers are rubber-stamping large changes. |
| High-risk large PR review depth (new subsystem / query engine) | X | >5 healthy / <2 gap | | Same as above, scoped to security/critical-path PRs. These warrant deeper scrutiny regardless of size. |
| Security review coverage | X% | % security-sensitive PRs with dedicated security skill invoked | | % of PRs touching auth/IAM/credentials/sandbox that had a dedicated security skill invoked. Low = security-sensitive changes merged without targeted review. |
| Rubber-stamp rate (high-blast-radius PRs, all approvers left 0 substantive comments) | X% | <10% healthy / 10-25% warning / >25% problem | | High-blast-radius PRs where every approver left zero substantive comments. Distinct from no-review: someone approved, but documented nothing they validated. |
| Bot-authored PR review depth (% with ≥1 substantive human comment) | X% | 100% target — every bot PR needs documented human validation | | % of AI/bot-authored PRs that had at least one substantive human comment. AI-generated code needs documented human validation before merge. |
| Revert/follow-up rate | X% | <5% healthy / 5-15% warning / >15% unstable | | % of PRs that reference a previous PR as a fix or follow-up. High = shipping incomplete/broken work and patching in follow-on commits. |
| Blast radius distribution | low=N med=N high=N | >50% low healthy / >30% high = risk | | Count of merged PRs per blast-radius level (low: ≤50 LOC+1 dir; medium: 51-300 LOC or 2 dirs; high: >300 LOC or 3+ dirs or sensitive paths). See `shared/merge-queue-metrics.md`. |
| Cross-scope PR rate | X% | <20% healthy / 20-40% warning / >40% coupling risk | | % of merged PRs touching 2+ CODEOWNERS entries or unrelated top-level directories. High = poor module boundaries or monolith coupling. |
| High-blast rubber-stamp rate | X% | 0% target / >0% = CRITICAL finding | | High-blast-radius PRs where ALL approvers left 0 substantive comments / total high-blast PRs. Distinct from overall rubber-stamp rate — scoped to highest-risk changes. |
| Sensitive path review coverage | X% | 100% target / <80% = HIGH gap | | % of PRs touching auth/security/billing/infra paths that had a dedicated security review (skill invocation or security-tagged reviewer). |
| Verbosity accumulation rate | X% | <10% healthy / 10-25% warning / >25% problem | | % of PRs where reviewers flagged verbosity signals (search comments for: "trivial", "delegates to", "wrapper", "redundant", "simplify") OR LOC delta >300 with no new tests. High = codebase is accumulating structural bloat across the sprint. |
| Complexity creep (large-fn PRs) | N | 0 ideal | | Count of PRs that introduced functions visibly >50 lines (proxy for CC>10) with no decomposition comment or follow-up ticket. Cross-reference with `shared/sloppiness-metrics.md` CC thresholds. |
| PRs analyzed | N | — | — | Sample size for all metrics above. |

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
    "ci_catch_rate": 0.0,
    "code_review_skill_catch_rate": 0.0,
    "bot_finding_follow_through_rate": 0.0,
    "human_review_burden": 0.0,
    "avg_pr_size_loc": 0,
    "large_pr_review_depth": 0.0,
    "security_review_invocation_rate": 0.0,
    "rubber_stamp_rate": 0.0,
    "revert_followup_rate": 0.0,
    "verbosity_accumulation_rate": 0.0,
    "complexity_creep_pr_count": 0
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
      "suggestion": "<specific change to make>",
      "pr_evidence_count": 0
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
