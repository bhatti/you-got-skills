---
name: ygs-review-deep
description: Deep multi-specialist PR review — routes changed files to 7 specialist dimensions in parallel, then synthesizes ranked findings. Use for high-risk PRs or large diffs.
argument-hint: "<pr-url-or-number>"
---

# Deep PR Review

For shared review protocol (severity classification, finding format, verdict mapping), read:
`~/.claude/skills/you-got-skills/skills/shared/review-scaffold.md`

This skill runs a 7-dimension specialist review in parallel, then synthesizes findings. For a standard 5-domain review, use `ygs-review-pr` instead.

---

## Reviewer stance

Review as a **principal engineer / architect** who must approve this change before it ships. You are not just checking for bugs — you are checking whether the change:

- Solves the right problem at the root, not a symptom
- Fits the existing architecture, conventions, and abstractions
- Is the simplest correct solution — not over-engineered, not under-specified
- Is production-grade: handles failure, concurrency, and scale
- Follows the principle of least surprise — a new reader can predict behavior from names and structure

Output findings as **feedback, suggestions, and questions** suitable for a PR comment. Every finding must:
- Include a `file:line` reference
- State **what** is wrong and **why it matters**
- Suggest a fix that explains **why** the fix is needed
- Be **verified** against the actual code — no false alarms, no hallucinations

Solve problems comprehensively. An approach-level flaw is more important than any line-level finding.

---

## Phase 0: Repo-local skill consolidation

Before loading any specialist, check for repo-local overrides per `shared/review-scaffold.md#repo-local-skill-consolidation`:

```bash
ls .claude/skills/ 2>/dev/null \
  | grep -E "security|architecture|sre|testing|performance|maintainability|logic|observability" \
  || echo "no repo-local overrides"
```

For each ACTIVE dimension where a repo-local file exists (e.g., `.claude/skills/security.md`):
- Read the repo-local file **first** (project-specific rules — takes priority on conflicts)
- **Also** read the ygs specialist below (universal baseline — fills any gap the repo file doesn't cover)
- Consolidate: repo wins on conflicts; ygs fills gaps; note the override in Informational

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
3. Apply the principal quality bar — verify the following are addressed:
   - **Performance at scale:** would the change hold up at millions of requests/sec or with large data? Flag anything that becomes a bottleneck or causes memory bloat at scale
   - **Concurrency:** new shared mutable state, lock contention across I/O, race conditions, connection pool exhaustion
   - **Architecture cohesion:** does the change fit the existing design, or does it introduce an inconsistent pattern? Check surrounding code before judging
   - **Data contract integrity:** API surfaces, wire formats, and error messages backward-compatible; blast radius documented
   - **Approach-level assessment:** is this the right solution, or a well-executed wrong direction? Was the change even necessary? Could it be simpler?
   - **Existing abstractions:** are there utilities, helpers, or patterns already in the codebase that this duplicates? Flag DRY violations; do not flag three similar lines as a violation if no second caller exists to justify the abstraction
   - **Principle of least surprise:** would a new reader be surprised by this code's behavior, naming, or structure?
4. Rank: MUST → SHOULD → MAY, then by file path within each tier
5. Write `reports/review.md` with a structured summary:
   - One-paragraph overall assessment (approach-level verdict first)
   - Table: dimension → finding count by severity
   - Full findings list, ranked as **feedback, suggestions, and questions**

---

## Verification gate

Before reporting any finding, verify it against the actual source (not just the diff hunk). Read surrounding context. Mark confidence: HIGH (provable) / MEDIUM (judgment call with evidence) / LOW (uncertain — explain why).

Never report a finding you have not verified. **No false positives. No hallucinations.** Double-check every finding — a wrong finding wastes reviewer time and erodes trust. When uncertain, frame it as a question rather than a stated defect.

---

Output ONLY this JSON on the last line:
`{"status":"DONE","findings_count":<N>,"verdict":"<APPROVE|REQUEST_CHANGES|COMMENT>","summary":"<one sentence>"}`
