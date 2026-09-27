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

## PR Categories

Domain category derived from **file paths** (authoritative when diffstat available), then **labels**, then **title/description** keywords. First match wins. Source of truth for all skills and scripts — do not inline these definitions elsewhere.

| Category | Blast Modifier | Key Path Signals | Key Labels | Title Keywords |
|----------|---------------|-----------------|------------|----------------|
| `test` | blast capped at **low** | /__tests__/, /test/, /tests/, .test.ts, .spec.ts, _test.go, test_*.py | test, sdet, qa | \[SDET\], \bSDET\b, \btest suite\b |
| `security` | +2 | auth, crypto, secret, credential, cert, tls, ssl | security, crypto, cve | \bsecurity\b, \bcve\b, \bvuln |
| `authn_authz` | +2 | authn, authz, oauth, iam, rbac, saml, sso, token, session | auth, authz, rbac | \bauth[nz]?\b, \bpermission\b, \baccess.control\b |
| `sre` | +1 | terraform, infra, k8s, kubernetes, helm, deploy, ansible, packer | terraform, infra, sre, ops | \bterraform\b, \binfra\b, \bk8s\b, \bdeploy\b |
| `data` | +1 | migration, schema, database, db/, sql, redis, kafka, etl | migration, database, schema | \bmigration\b, \bschema\b, \bdatabase\b |
| `api` | +0 | api/, route, handler, controller, endpoint, grpc, proto | api, grpc | \bapi\b, \bendpoint\b, \broute\b |
| `ui` | +0 | frontend, web/, ui/, component, .tsx, .vue, .svelte, .css, .scss | frontend, ui, ux | \bui\b, \bfrontend\b, \bcomponent\b |
| `config` | +0 | config, .yaml, .yml, .toml, .env, settings | config, configuration | \bconfig\b, \bsettings\b |
| `backend` | +0 | src/, pkg/, lib/, service, core/ | (none — catch-all) | (none) |
| `unknown` | +0 | no signal matched | | |

**Blast modifier semantics:** add the modifier to the blast_radius dimension score (0–10 scale) before computing the composite risk score. A PR with blast_radius=medium (score=5) and category=security (+2) gets blast dimension score=7.

**Blast caps for special PR types** (applied by `collect_ready.py` before analysis — use `blast_radius` field directly as ground truth):
- `is_test_pr=true` → blast_radius capped at **low** (test-only PRs have zero production impact regardless of file count)
- `is_docs_pr=true` → blast_radius capped at **low** (docs/assets PRs do not change production behaviour)
- `is_wip_pr=true` → blast_radius capped at **medium** (WIP/draft PRs are not yet ready for high-risk merge lanes)

These caps are data-driven: `is_test_pr` is set when ≥80% of changed files match test-file path patterns, OR the title/branch contains SDET/QA patterns. `is_docs_pr` is set when all changed files are docs/markdown/assets. Do NOT re-derive from title keywords — trust the pre-computed fields.

**Hotspot detection:** if ≥3 `bug`-type PRs in the current queue share the same `category`, that category is a **hotspot** — flag it prominently. Hotspots indicate an area with elevated defect density that warrants extra scrutiny.

**PR type classification** (labels first, then title keywords):
- `bug`: label contains `bug`, `fix`, `hotfix`, `defect` — OR title matches `\b(fix|bug|hotfix|patch|defect|regression|crash)\b`
- `feature`: label contains `feature`, `feat`, `story`, `enhancement` — OR title matches `\b(feat|feature|story|enhancement|implement|add)\b`
- `unknown`: no signal matched

**Category confidence:**
- `file_path` — derived from actual changed file paths (diffstat); most reliable
- `label` — derived from PR labels
- `title` — derived from PR title/description text; least reliable
- `unknown` — no data available (e.g., diffstat API failed)

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
