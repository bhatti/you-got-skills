---
name: ygs-merge-queue
description: "Read-only merge queue analysis: group open PRs by scope lane, score blast-radius and risk, flag PRs needing human review. Works for a single repo (GitHub or Bitbucket). No PR mutations."
argument-hint: "[--repo <org/repo>] [--target-branch <branch>] [--target <branch>] [--label <filter>]"
---

# Merge Queue Analysis

Read `~/.claude/skills/you-got-skills/skills/shared/merge-queue-metrics.md`.
Read `~/.claude/skills/you-got-skills/skills/shared/pr-audit-context.md`.
Read `~/.claude/skills/you-got-skills/skills/shared/output-format.md`.
Read `~/.claude/skills/you-got-skills/skills/shared/tracker.md`.

**READ-ONLY**: this skill never merges, labels, or comments on PRs. Pure analysis only.

## Step 1: Load pre-fetched data

`lane_groups.json` and `ready_prs.json` are already in `/workspace`. Read them. Do NOT make API calls.

```bash
cat /workspace/ready_prs.json    # {pr_count, repo, target_branch_filter, prs: [{pr_number, title,
                                  #   author, scope, blast_radius, pr_type, age_hours, branch,
                                  #   target_branch, ci_status (none=unavailable for BB),
                                  #   has_approval, approval_count, reviewer_count,
                                  #   issue_ref ({key,url} or null), url, labels}]}
cat /workspace/lane_groups.json  # {lanes: [{lane_id, prs: [...], category_counts, hotspots}]}
```

If either file is missing or empty, report: "No open PRs found for `{repo}`. Either the repo has no open PRs or the collector failed." and exit `{"status":"DONE","total_prs":0}`.

If `ready_prs.json` contains a non-empty `target_branch_filter`, acknowledge it in the report header:
`Analysis scoped to PRs targeting: **{target_branch_filter}**`

Use `shared/pr-audit-context.md` for the per-PR data format definition and bot-detection rules — this ensures consistent field interpretation with `ygs-pr-audit`.

Note the `repo` field in `ready_prs.json` — show it in the report for context.

## Step 2: Score each PR

**`blast_radius`, `category`, and `pr_type` are pre-computed from actual file paths and labels — do NOT re-derive them. Use them as ground truth.**

**Blast caps are pre-applied**: `is_test_pr`, `is_docs_pr`, `is_wip_pr` flags are set by the collector and blast_radius is already capped (test/docs → low, wip → medium). Do not add category modifiers for test PRs. The `Type` column shows 🚧 for WIP and 📝 for docs PRs as visual indicators.

Apply the canonical 6-dimension risk model from `shared/merge-queue-metrics.md#risk-dimensions`. Risk tiers: LOW 0–15, MEDIUM 16–30, HIGH 31+.

**Category blast modifier** (from `shared/merge-queue-metrics.md#pr-categories`): add to blast_radius dimension score before computing composite:
- `security` or `authn_authz`: +2
- `sre` or `data`: +1
- all others: +0

Readiness gate signals (applied on top of the risk score):

| Signal | Green (🟢) | Yellow (🟡) | Red (🔴) |
|--------|-----------|------------|---------|
| blast_radius | low | medium | high |
| ci_status | success / none | pending | failed |
| has_approval | true | — | false (age > 24h) |
| age_hours | < 48h | 48–168h | > 168h (1 week) |
| pr_type | feature/unknown | — | bug (prioritize review) |
| category | api/ui/config/backend | data/sre | security/authn_authz (⚠️ always flag) |

**Human review required** when: `blast_radius = high` OR `ci_status = failed` OR `category` in {security, authn_authz}.

**Hotspot detection** (from `shared/merge-queue-metrics.md#pr-categories`): if `lane_groups.json` has `"hotspots": [...]` for a lane, flag prominently — this means ≥3 bug PRs share that category, indicating elevated defect density. If `category_confidence = "unknown"`, note the classification is based on limited data.

## Step 3: Detect conflict risk per lane

PRs in the same lane share a broad scope by design. Conflict risk within a lane means
two or more PRs may be modifying **overlapping specific paths**. Use `branch` names as
a proxy: if two branches in the same lane share a descriptive path prefix
(e.g. `fix/billing-invoices-*` and `feat/billing-invoices-refactor`), flag HIGH.

| Condition | Conflict Risk |
|-----------|--------------|
| Lane has ≤1 PR | NONE |
| PRs have clearly disjoint branch prefixes or unrelated names | NONE |
| 2+ PRs share a branch path prefix within the same lane | HIGH |
| Branch names are non-descriptive (e.g. `fix/123`) — cannot determine | NONE (conservative) |

## Step 4: Build hierarchical branch → risk-tier → per-PR report

**CI note**: If all `ci_status` values are `"none"`, add this note once at the top of the lanes section:
> ℹ️ **CI status: N/A** — Bitbucket REST API does not return pipeline status in the bulk PR list endpoint.

**Report structure** — hierarchical: branch → risk tier → per-PR table:

```
### Branch: stage (N PRs — 🔴 N high, 🟡 N medium, 🟢 N low)

#### 🔴 High Risk (N PRs)
Hotspots: {categories with ≥3 bug PRs in this tier, or "none"}
Conflict risk: NONE / ⚠️ HIGH

| PR | Title | Category | Type | Blast | CI | Age | Reviewers | Issues |
|----|-------|----------|------|-------|----|-----|-----------|--------|
| [#42](url) | Fix billing calc | api | 🐛 | 🟡 medium | ✅ | 2h | 1/2 ✅ | [BILL-123](url) |
| [#38](url) | Auth refactor | ⚠️ authn_authz | ✨ | 🔴 high | N/A | 3d | 0/3 ✅ | — |

**PR details:**
- [#42](url) [BILL-123](url): Fix billing calculation rounding error — api 🐛
- [#38](url): Refactor auth token refresh flow — authn_authz ✨
```

**Columns:**
- **PR**: `[#{num}](url)` link
- **Title**: truncated to ~50 chars
- **Category**: `⚠️` prefix for `security`, `authn_authz`; asterisk `*` suffix when `category_confidence` is `title` (heuristic)
- **Type**: `pr_type` emoji — 🐛 bug, ✨ feature, ❓ unknown
- **Blast**: `🔴/🟡/🟢 {level}`
- **CI**: `✅` success, `❌` failed, `⏳` pending, `N/A` when unavailable (all-none), `—` otherwise
- **Age**: `{N}h` for <24h, `{N}d` for ≥24h
- **Reviewers**: `{approvals}/{total} ✅` — from `approval_count` / `reviewer_count`; `—` when no reviewers assigned
- **Issues**: `[KEY](url)` from `issue_ref.key` + `issue_ref.url`; `—` when none

**After each tier's table**, add `**PR details:**` list with full title + issue ref (no truncation).

**Stacked PRs** get a separate section with an extra **Target** column showing the feature branch they target.

`pr_type` emoji: 🐛 = bug, ✨ = feature, ❓ = unknown.
Category ⚠️ prefix for: `security`, `authn_authz`, `sre`, `data`.
Highest risk per branch = max blast_radius across all tiers (high > medium > low).

## Step 5: Format final report and output JSON

Follow `shared/output-format.md` for Slack-compatible formatting.

**Report header:**
```
## Merge Queue Analysis — {repo}
{total_prs} open PRs across {lane_count} lanes
```

Then per-lane tables from Step 4.

**Summary section at end:**
```
### Summary
- 🟢 Low risk: {n} PRs
- 🟡 Medium risk: {n} PRs  
- 🔴 High risk: {n} PRs (needs human review)
- ⚠️ Conflict risk: {n} lanes
```

**Recommendations** — one line per lane:
- All low blast + CI passing → "Safe to batch merge"
- Any high blast or ci_status=failed → "Block: human review required for PR #{n}"
- Conflict detected → "Conflict risk: merge {lane_id} PRs one at a time"

**Exit JSON (last line of output):**
```json
{"status":"DONE","repo":"{repo}","total_prs":N,"lanes":N,"high_risk_prs":N,"needs_human_review":N,"conflict_lanes":N}
```

Use `DONE_WITH_CONCERNS` if any PR needs human review or any lane has conflict risk.

---

**Shared refs:** `shared/merge-queue-metrics.md` (blast radius, risk tiers), `shared/output-format.md` (Slack format), `shared/tracker.md` (GH vs BB resolution)

**Does NOT read:** `shared/risk-criteria.md` (that's sprint delivery risk — different model)

**Cross-skill refs:** `ygs-pr-audit` for per-PR deep dive; `ygs-scope-router` for authoritative CODEOWNERS scope; `ygs-risk-scan` for sprint-level delivery risk
