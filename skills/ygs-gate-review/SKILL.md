---
name: ygs-gate-review
description: "Read-only single-PR gate review: blast radius, risk score, AI findings — no PR mutations. Alias: @bot scope <pr>."
argument-hint: "<pr-url-or-number>"
---

# Gate Review — Scope & Blast Radius

Read `~/.claude/skills/you-got-skills/skills/shared/merge-queue-metrics.md`.
Read `~/.claude/skills/you-got-skills/skills/shared/review-scaffold.md`.

**READ-ONLY**: never comments on, labels, or mutates a PR. Pure gate analysis.

**Aliases**: `@bot gate-review <pr>` and `@bot scope <pr>` invoke this same workflow.

## Input data

Pre-fetched artifacts are in `/workspace`:

```bash
cat /workspace/scope.json       # {scope, blast_radius, changed_files, additions, deletions,
                                 #  owners, sensitive_touched, categories, author, created_at}
cat /workspace/risk_score.json  # {score, tier, dimensions: [{name, score, weight, evidence}]}
cat /workspace/review_result.json  # {findings: [{severity, description, file, line}]}
```

If `scope.json` is missing, report: "Scope data unavailable — clone or scope_router step may have failed." and exit.

## Report structure

Produce a single Markdown report with:

### 1. Risk Score

Show tier (🟢 LOW / 🟡 MEDIUM / 🔴 HIGH), composite score, and the risk dimension breakdown table:

```
| Dimension | Score | Weight | Evidence |
```

Apply the 6-dimension model from `shared/merge-queue-metrics.md#risk-dimensions`.

### 2. Scope & Blast Radius

Show the scope table with ALL available fields from `scope.json`:

```
| Field | Value | Description |
```

Always include: Scope, Blast radius, Changed files, Lines changed, Owners.
Include when present: LOC added, LOC deleted, File categories, Sensitive paths, PR author, PR date.

### 3. Review Findings

If `review_result.json` has findings: list them by severity (MUST → SHOULD → MAY).
If empty findings: `✅ No issues found`.

## Output format

Follow `shared/output-format.md` for Slack-compatible Markdown.
The HTML report artifact is rendered from this Markdown by `report.py` — do not add HTML tags.

## Scope

- Analyze the single PR identified by `PR_NUMBER` env var.
- Do NOT re-derive blast_radius or scope — use pre-computed values from `scope.json` as ground truth.
- Risk scoring uses `shared/merge-queue-metrics.md#risk-dimensions`; the pre-computed `risk_score.json` is authoritative — show it, do not recompute.

**Shared refs:** `shared/merge-queue-metrics.md` (blast radius, risk tiers), `shared/output-format.md` (Slack format), `shared/tracker.md` (GH vs BB)
