---
name: ygs-review-deep
description: Deep multi-specialist PR review — routes changed files to 7 specialist dimensions in parallel, then synthesizes ranked findings. Use for high-risk PRs or large diffs.
argument-hint: "<pr-url-or-number>"
---

# Deep PR Review

For shared review protocol (severity classification, finding format, verdict mapping), read:
`~/.claude/skills/you-got-skills/skills/shared/review-scaffold.md`

This skill runs a 7-dimension specialist review in parallel, then synthesizes findings. For a standard 4-domain review, use `ygs-review-pr` instead.

---

## Phase 1: Get diff and route dimensions

Follow the diff protocol from `shared/review-scaffold.md`. Then classify each changed file across 7 dimensions:

| Dimension | Specialist file | Mark ACTIVE when... |
|-----------|----------------|---------------------|
| Logic | `specialists/logic.md` | Any new logic, conditionals, error paths |
| Security | `specialists/security.md` | Auth, data handling, external input, new endpoints |
| Architecture | `specialists/architecture.md` | New modules, imports, abstractions, cross-module changes |
| Performance | `specialists/performance.md` | Loops, DB queries, network calls, caches, hot paths |
| Testing | `specialists/testing.md` | Any test file added or changed; any logic without test changes |
| Observability | `specialists/observability.md` | Logging, metrics, tracing, error handling, new code paths |
| Maintainability | `specialists/maintainability.md` | All changes (always ACTIVE — naming and dead code apply everywhere) |

Mark a dimension SKIP only if the diff clearly has no bearing on it (e.g., a config comment change skips Logic, Performance, Security).

---

## Phase 2: Run active specialist passes in parallel

Load each ACTIVE specialist reference file and review the diff through that lens simultaneously. Each pass is focused — do not repeat findings from another dimension.

Read the relevant specialist reference files:
- `~/.claude/skills/you-got-skills/skills/ygs-review-deep/specialists/logic.md`
- `~/.claude/skills/you-got-skills/skills/ygs-review-deep/specialists/security.md`
- `~/.claude/skills/you-got-skills/skills/ygs-review-deep/specialists/architecture.md`
- `~/.claude/skills/you-got-skills/skills/ygs-review-deep/specialists/performance.md`
- `~/.claude/skills/you-got-skills/skills/ygs-review-deep/specialists/testing.md`
- `~/.claude/skills/you-got-skills/skills/ygs-review-deep/specialists/observability.md`
- `~/.claude/skills/you-got-skills/skills/ygs-review-deep/specialists/maintainability.md`

For each active dimension, collect findings tagged `[DIM]` (e.g., `[SECURITY]`, `[PERF]`).

---

## Phase 3: Synthesize

1. Merge all findings across active dimensions
2. Deduplicate: if two dimensions flag the same file+line for the same reason, keep the higher-severity finding, note the secondary dimension in parentheses
3. Apply the principal quality bar from `shared/review-scaffold.md` — verify performance-at-scale, concurrency, architecture cohesion, and data contract checks are addressed
4. Rank: MUST → SHOULD → MAY, then by file path within each tier
5. Write `reports/review.md` with a structured summary:
   - One-paragraph overall assessment (approach-level verdict first)
   - Table: dimension → finding count by severity
   - Full findings list, ranked

---

## Verification gate

Before reporting any finding, verify it against the actual source (not just the diff hunk). Read surrounding context. Mark confidence: HIGH (provable) / MEDIUM (judgment call with evidence) / LOW (uncertain — explain why).

Never report a finding you have not verified. No false positives.

---

Output ONLY this JSON on the last line:
`{"status":"DONE","findings_count":<N>,"verdict":"<APPROVE|REQUEST_CHANGES|COMMENT>","summary":"<one sentence>"}`
