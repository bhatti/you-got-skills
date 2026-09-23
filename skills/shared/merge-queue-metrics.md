# Shared Reference: Merge Queue & PR Risk Metrics

Single source of truth for blast radius, scope classification, risk dimensions, and merge-readiness
signals. Referenced by MQ scripts (`scripts/mq/scope_router.py`, `scripts/mq/risk_score.py`,
`scripts/mq/report.py`) and audit skills (`ygs-pr-audit`, `ygs-scope-router`).

**Do not duplicate definitions in individual skills** — reference this file instead.

---

## Scope Classification

| Scope Key | Definition | Example |
|-----------|-----------|---------|
| `<team-name>` | All changed files owned by a single CODEOWNERS entry | `billing-service` — all files under `src/billing/` owned by `@team-payments` |
| `<directory>` | All changed files under one top-level directory (no CODEOWNERS) | `api` — all files under `api/` |
| `cross-scope` | Files span 2+ CODEOWNERS entries or 2+ unrelated top-level directories | `auth/` + `billing/` + `frontend/` touched in one PR |
| `empty` | No changed files detected | PR with only metadata changes |

**Why scope matters:** Independent scopes can be tested and merged in parallel (separate merge queue lanes). Cross-scope PRs must serialize because a failure could be caused by interaction between modules.

---

## Blast Radius

| Level | Criteria | Description | Merge Queue Impact |
|-------|----------|-------------|-------------------|
| **low** | ≤50 lines changed AND 1 module | Small, contained change — unlikely to break unrelated code | Fast lane: can batch with other low-blast PRs |
| **medium** | 51–300 lines OR 2 modules | Moderate change — could affect adjacent modules | Standard lane: tested independently before merge |
| **high** | >300 lines OR 3+ modules OR touches sensitive paths | Large or security-sensitive change — wide failure surface | Requires human approval; tested in isolation |

**Sensitive paths** (trigger high blast radius regardless of size — from `_shared.py:SENSITIVE_PATHS` regex):
`auth|security|billing|payments|crypto|secrets|credentials|.env|migrations|rbac|iam|oauth|tokens`

---

## Risk Dimensions

Each dimension scored 0–10. Higher = more risk.

| Dimension | Key | Weight | Score Ranges | What It Measures |
|-----------|-----|--------|-------------|-----------------|
| **Size** | `size` | 1.5× | 1 (≤20 LOC) → 10 (>1000 LOC) | Total lines changed (additions + deletions). Larger PRs have more surface area for defects and are harder to review thoroughly. |
| **File Count** | `file_count` | 1.0× | 1 (≤3 files) → 10 (>50 files) | Number of files modified. More files = more integration points that could break. |
| **Blast Radius** | `blast_radius` | 2.0× | 2 (low) / 5 (medium) / 9 (high) | How many modules and teams are affected. Cross-scope changes multiply the chance of unexpected interaction failures. |
| **Sensitive Paths** | `sensitive_paths` | 2.5× | 0 (none) → 10 (>5 sensitive files) | Files touching auth, security, billing, infrastructure, or deployment. Defects here have outsized production impact. |
| **Test Coverage** | `test_coverage` | 1.5× | 0 (≥1:1 test:source ratio) → 8 (0 test files) | Ratio of test files to source files in the PR. No tests = changes are unverified; defects ship silently. |
| **Historical** | `historical` | 1.0× | 1 (≤1% defect rate) → 9 (>10% defect rate) | Recent defect rate for the changed paths. Areas with frequent past defects are likely to produce more. Default: 3 (no history available). |

**Composite formula:** `Σ(dimension_score × weight)` — weighted sum, not average.

---

## Risk Tiers

| Tier | Score Range | Emoji | Meaning | Merge Queue Action |
|------|------------|-------|---------|-------------------|
| **LOW** | 0–15 | 🟢 | Safe to auto-merge after CI passes | Batch with other low-risk PRs for speculative merge |
| **MEDIUM** | 16–30 | 🟡 | Needs standard review; acceptable risk | Test independently; one reviewer approval sufficient |
| **HIGH** | 31+ | 🔴 | Requires human approval before merge | Block merge until designated reviewer signs off; test in isolation |

---

## Merge Readiness Signals

Used by `collect_ready.py` and `group_by_scope.py` to evaluate queue eligibility.

| Signal | Check | Blocks Queue Entry? |
|--------|-------|-------------------|
| CI green | All required checks passing | Yes — never queue a red PR |
| Review approved | At least one non-author approval | Yes for high-risk; No for low-risk |
| No unresolved threads | All review conversations resolved | Yes |
| Scope labeled | `scope:X` label present | No — auto-computed if missing |
| Risk assessed | `risk_score.json` exists | No — auto-computed if missing |
| Batch eligible | risk < 40 AND changed_files < 50 | No — determines batching strategy only |

---

## PR Audit Integration

When `ygs-pr-audit` runs, compute these per-PR metrics from the pre-computed data and
include them in the **Metrics Dashboard** and **Design Gaps** specialist:

| Metric | Formula | Benchmark | Use in Audit |
|--------|---------|-----------|-------------|
| Blast radius distribution | Count PRs per blast-radius level | >50% low = healthy | Flag if >30% of merged PRs are high blast radius |
| Cross-scope PR rate | cross-scope PRs / total PRs | <20% healthy | High rate = poor module boundaries or too-large PRs |
| High-risk merge rate | HIGH-tier PRs merged without extra review / total HIGH PRs | 0% target | Any high-risk PR merged with rubber-stamp = CRITICAL finding |
| Sensitive path coverage | sensitive-path PRs with security skill invoked / total sensitive PRs | 100% target | Gap = security review process missing |
| Size → review depth correlation | avg comments on >400 LOC PRs vs ≤400 LOC | large > small | Flat = review depth doesn't scale with risk |

---

## Report Table Formats

### Scope Table (always include Description column)

```
| Field | Value | Description |
|-------|-------|-------------|
| Scope | **{scope}** | {scope_description} |
| Blast radius | **{blast}** | {blast_description} |
| Changed files | {n} | Number of files modified in this PR |
| Lines changed | {n} | Total additions + deletions |
| Sensitive paths | `path1`, `path2` | Files matching auth/security/billing/infra patterns |
```

### Risk Table (always include Description column)

```
| Dimension | Score | Weight | Description |
|-----------|-------|--------|-------------|
| Size | {n}/10 | 1.5× | {size_description} |
| File Count | {n}/10 | 1.0× | {file_count_description} |
| ... | ... | ... | ... |
```
