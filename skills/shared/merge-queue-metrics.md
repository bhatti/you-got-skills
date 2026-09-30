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

**PR type classification** (security title keywords first, then labels, then title keywords, then flags):
- `security`: title/description matches `\b(vulnerability|cve|rce|ssrf|xss|injection|exploit|0-?day|zero.?day|security.?fix)\b` — always wins regardless of labels (a vuln fix labelled 'bug' is still a security fix for risk scoring)
- `bug`: label contains `bug`, `fix`, `hotfix`, `defect` — OR title matches `\b(fix|bug|hotfix|hot.?fix|patch|defect|regression|crash|revert|rollback|roll.?back|roll.?forward|workaround|broken)\b`
- `feature`: label contains `feature`, `feat`, `story`, `enhancement` — OR title matches `\b(feat|feature|story|enhancement|implement|add)\b`
- `refactor`: label contains `refactor`, `cleanup`, `tech-debt` — OR title matches `\b(refactor|cleanup|detangle|extract|reorganize|restructure|simplify|split|rename|move)\b`
- `chore`: label contains `chore`, `deps`, `dependency`, `maintenance` — OR title matches `\b(chore|deps?|dependency|upgrade|bump|update|version|migrate)\b`
- `test`: `is_test_pr=true` flag (from file paths or title patterns)
- `docs`: `is_docs_pr=true` flag (from file paths or title patterns)
- `feature`: fallback when no other signal matches (was `unknown` — removed to avoid misleading "unknown type" audit findings)

**Defect rate proxy** (connects to Joe Magerramov's model — see blog Part 1.3):
- Defect PRs = `bug` + `security` type PRs
- Defect rate proxy = defect PRs / total PRs (maps to per-commit defect probability)
- Feature:Bug ratio = feature PRs / defect PRs (healthy ≥ 3:1, concerning < 1:1)
- Batch success estimate = (1 - defect_rate)^batch_size

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

## Deployment Risk Model

Extends Joe Magerramov's merge-batch formula to the full deployment pipeline. All formulas are canonical in `scripts/mq/simulate.py` — report.py imports them; do not duplicate.

### Release Train Success

Most orgs don't use CD — they batch PRs into daily/weekly release trains. The formula is the same as merge-batch success but applied at the release level:

```
release_success = (1 - defect_rate) ^ prs_per_release
```

CD is the special case where `prs_per_release = 1`. A weekly train of 50 PRs at 2% defect rate has only 36% chance of shipping clean.

### Rollback Feasibility

When a bug in release R1 is found after R2, R3, R4 are deployed, rolling back requires reverting all subsequent releases too:

| Stacked Releases | Strategy | MTTR Multiplier | Reason |
|-----------------|----------|----------------|--------|
| ≤1 | Rollback | 1.0× | Single release — clean rollback is straightforward |
| 2–3 | Rollback | 1.5× | Must revert intermediate releases — costly but feasible |
| ≥4 | Roll-forward | 2.5× | Too many intermediate releases — rollback is impractical |

### Deployment Profiles

Single env var `DEPLOYMENT_PROFILE` — either a preset name or path to a JSON file.

**Presets** (built into `simulate.py`):

| Preset | Release Cadence | PRs/Release | Stacked Releases | Maturity Level |
|--------|----------------|-------------|-----------------|----------------|
| `cd` | Continuous | 1 | 1 | Advanced (high across all dimensions) |
| `daily-train` | Daily batch | queue_size / 5 | 2 | Intermediate |
| `weekly-train` | Weekly batch | queue_size | 3 | Foundational–Intermediate |
| `manual` | Ad-hoc | queue_size | 5 | Foundational (minimal automation) |

**Custom profile JSON schema:**
```json
{
  "release_cadence": "weekly-train",
  "prs_per_release": 50,
  "releases_stacked": 3,
  "maturity": {
    "automated_testing": 0.8,
    "canary_deployment": 0.0,
    "automated_rollback": 0.0,
    "observability": 0.5,
    "wave_deployment": 0.0,
    "feature_flags": 0.3,
    "blue_green": 0.0
  }
}
```

---

## Deployment Maturity Dimensions

7 weighted dimensions, each scored 0.0 (absent) to 1.0 (fully implemented). Max weighted score = 9.5.

| Dimension | Weight | What It Enables |
|-----------|--------|-----------------|
| `automated_testing` | 2.0 | CI on every PR catches defects pre-merge |
| `canary_deployment` | 2.0 | Progressive rollout catches prod-only failures |
| `automated_rollback` | 1.5 | Instant revert on anomaly detection |
| `observability` | 1.5 | Alerting + metrics detect failures fast |
| `wave_deployment` | 1.0 | Stage → preprod → prod progression |
| `feature_flags` | 1.0 | Decouple deploy from release |
| `blue_green` | 0.5 | Zero-downtime deploy infrastructure |

**Maturity tiers:**

| Tier | Score Range | Effective Risk Multiplier | Description |
|------|------------|--------------------------|-------------|
| Foundational | 0–3.9 | 0.71–1.0 | Minimal automation — defects hit production at full force |
| Intermediate | 4–6.9 | 0.49–0.71 | Partial automation — some defects caught before users see them |
| Advanced | 7–9.5 | 0.30–0.48 | Full pipeline — canary + rollback + observability catch most defects |

**Risk multiplier formula:** `max(0.3, 1.0 - maturity_score / 13.585)` — maturity reduces effective deployment risk by up to 70%.

---

## Blue Line / Red Line

The MQ report shows a text gauge with the current position (🔵 "blue line") vs the calamity threshold (🔴 "red line").

**Calamity threshold formula:**
```
max_safe_batch = log(target_success) / log(1 - defect_rate)
```

Default `target_success = 0.70` — below this, more than 30% of releases contain a defect.

**Gauge rendering:**
```
[🟢🟢🟢🟢🟢🔵░░🔴░░░░░░]  82% success | calamity at 18 PRs/batch
```

**Zone thresholds:**

| Zone | Success Rate | Meaning |
|------|-------------|---------|
| 🟢 Green | ≥ 90% | Healthy — well within safe operating range |
| 🟡 Yellow | 70–89% | Warning — approaching the cliff |
| 🔴 Red | < 70% | Danger — past the calamity threshold; more than 30% of releases will contain defects |

**Headroom:** `(max_safe_batch - current_batch) / max_safe_batch × 100` — how far current batch size is from the cliff as a percentage. At 2% defect rate, the red line is at ~18 PRs/batch.

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

### Risk Table (always include Evidence column)

```
| Dimension | Score | Weight | Evidence |
|-----------|-------|--------|----------|
| Size | {n}/10 | 1.5× | {size_description} |
| File Count | {n}/10 | 1.0× | {file_count_description} |
| ... | ... | ... | ... |
```

### Metrics Dashboard (5-column — canonical across all reports)

All reports (pr-audit, mq, gate-review/scope, code-audit) use this exact format. Generated by
`build_metrics_dashboard()` in `scripts/common/pr_classify.py` — do not hand-roll in skills.

```
| Metric | Value | Benchmark | Signal | Description |
|--------|-------|-----------|--------|-------------|
| PR Volume | {n} PRs | — | — | Total PRs analyzed |
| Defect Rate (proxy) | {n}% | <10% healthy | 🟢/🟡/🔴 | Bug+security PRs / total |
| Feature:Bug Ratio | {n}:{m} | ≥3:1 healthy | 🟢/🟡/🔴 | Work type balance |
| Blast Radius Distribution | {low}L/{med}M/{high}H | high≤10% | 🟢/🟡/🔴 | Scope concentration |
| CI Pass Rate | {n}% | ≥90% | 🟢/🟡/🔴 | CI green rate (N/A if all unknown) |
| {report-specific rows via extra_rows} | ... | ... | ... | ... |
```

**Extra rows by report type:**
- `ygs-pr-audit`: date range (first merged → last merged), stale PRs (>14d)
- `ygs-codebase-audit`: date range (oldest → head commit), files analyzed, active authors, revert commits
- `ygs-merge-queue`: same 5-column format from pre-computed lane data

---

## DORA and Throughput Metrics

These metrics are computed by `compute_throughput_metrics(prs)` in `scripts/common/pr_classify.py` when ≥3 merged PRs are available. They appear as DORA rows in `build_metrics_dashboard()`.

| Metric | Key | Benchmark | Computability | Notes |
|--------|-----|-----------|---------------|-------|
| Deployment Frequency | `deployment_frequency` | ≥5/wk elite | ✅ from `merged_at` | Merged PRs per week |
| Lead Time (P50) | `lead_time_p50_days` | ≤1d elite, ≤7d high | ✅ from `created_at`→`merged_at` | Median time from PR open to merge |
| Change Failure Rate | `change_failure_rate_pct` | ≤10% healthy | ✅ from `pr_type` | Bug+security PRs as % of total merged |
| PR Survival Rate | `pr_survival_rate_pct` | ≥80% healthy | ✅ from `state` | Merged/(merged+declined) |
| Review Lag | (not computed) | — | ❌ requires `reviews[].submittedAt` not fetched | Would need per-PR review API call |
| MTTR | (not computed) | — | ❌ no incident data | No incident/outage datasource |

**DORA tier mapping (Lead Time):**
- Elite: <1 day
- High: <1 week
- Medium: <1 month
- Low: >1 month

**CFR Proxy:** Uses PR type to proxy change failure rate. `pr_type=bug` or `pr_type=security` = failure. This underestimates CFR since bugs found in production (hotfixes) may be labeled differently.

---

## Slack Output Format

The MQ report (`report.py`) posts two separate outputs:

**Slack summary (~800 chars, condensed)** — posted as thread message:
```
*Merge Queue* — YYYY-MM-DD → YYYY-MM-DD
*Queue Health*: {N} open PRs · ✅/🔴 {risk_signal} · 🔴 {high_blast} high-blast · 🔥 {hotspots} hotspot paths
*Review*: Review: {rev_pct}% ✅/🔴 · CI: {ci_pass}/{ci_known} ({ci_pct}%) ✅/🔴 · 🔴 {stale_7d} stale (>7d)
*Risk*: ⚠️ {needs_review} need human review · 🔀 {conflict_lanes} conflict lane(s) · 🕐 {stale_14d} stale (>14d)
*Work*: feature:{n} · bug:{n} · test:{n} · refactor:{n}
*Throughput*: {avg_age}d avg age · CFR proxy: {cfr}% ✅/🔴
Full report in thread ↑
```

**Full report** (HTML artifact) — attached as file in thread:
Complete lane tables, Valley of Calm analysis, deployment risk, risk heatmap.
