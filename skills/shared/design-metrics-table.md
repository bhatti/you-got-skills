# Design Quality Metrics Table — Canonical Format

Every skill that analyzes design quality **must** emit this table in its report output.
The schema is fixed — all 8 rows, same column order, same signal thresholds.
Write `N/A` for rows not applicable at a given scope (e.g., per-PR skills write N/A for aggregate rows). In `findings.json` a value you did not assess is `null`, never `0`: the report renders it as `—` / "not assessed" (a reported `0` is a clean result).

**Column definitions** (matches the 5-column Metrics Dashboard from `shared/quality-checklist.md` §9):

| Column | Content |
|--------|---------|
| Principle | Fixed row name — do not rename |
| Score | Count of violations (per-PR) or rate/average (aggregate) |
| Benchmark | The healthy target |
| Signal | 🟢 healthy / 🟡 warning / 🔴 concern |
| Notes | Specific file:line evidence or trend note |

---

## Canonical Table Template

Copy this verbatim into the report output section. Replace `<…>` with actual values.

```markdown
## Design Quality Metrics

| Principle | Score | Benchmark | Signal | Notes |
|-----------|-------|-----------|--------|-------|
| SRP violations | <N> | 0 ideal | <🟢/🟡/🔴> | <file:line or "none found"> |
| DIP violations | <N> | 0 ideal | <🟢/🟡/🔴> | <concrete dep in domain, or "none found"> |
| Cyclic deps (ADP) | <N> | 0 | <🟢/🔴> | <cycle pair or "none detected"> |
| Layer violations | <N> | 0 ideal | <🟢/🟡/🔴> | <wrong-direction import or "none"> |
| Avg instability (I) | <0.x> | < 0.5 (core) | <🟢/🟡/🔴> | <module name or "N/A"> |
| Hotspot coupling | <N files> | 0 ideal | <🟢/🟡/🔴> | <file name or "none"> |
| Readability score | <N>/10 | > 7 | <🟢/🟡/🔴> | <weakest sub-dimension or "all good"> |
| ISP / fat interface | <N> | 0 ideal | <🟢/🟡/🔴> | <interface name or "none found"> |
```

---

## Signal Thresholds (apply uniformly across all skills)

| Row | 🟢 Healthy | 🟡 Warning | 🔴 Concern |
|-----|-----------|-----------|-----------|
| SRP violations | 0 | 1–2 | 3+ |
| DIP violations | 0 | 1 | 2+ |
| Cyclic deps (ADP) | 0 | — | Any |
| Layer violations | 0 | 1–2 | 3+ |
| Avg instability (I) | < 0.5 | 0.5–0.7 | > 0.7 |
| Hotspot coupling | 0 | 1–2 | 3+ |
| Readability score | > 7 | 5–7 | < 5 |
| ISP / fat interface | 0 | 1 | 2+ |

---

## Scope Interpretation by Skill

| Skill | Score column meaning | Notes column |
|-------|---------------------|-------------|
| `ygs-pr-risk` | Count of violations in diff | Cite file:line from diff |
| `ygs-code-review` | Count of violations in diff | Cite file:line from diff |
| `ygs-review-deep` | Count across all active dimensions | Dimension tag in parentheses |
| `ygs-pr-audit` | Rate per PR across N PRs analyzed | "X of N PRs had this violation" |
| `ygs-codebase-audit` | Total count across whole codebase | Top offending file |

**Instability (I) special cases:**
- Per-PR skills: estimate I only for modules directly changed; write `N/A` if changed files are leaf adapters (expected I ≈ 1.0)
- Aggregate skills: compute mean I across all non-leaf, non-test modules

**Readability score special cases:**
- Per-PR skills: average score across changed functions only (not unchanged context)
- Aggregate skills: if readability was not scored per-PR, write `N/A` (do not fabricate a number)

---

## LSP, OCP, CCP, CRP, REP — findings list only

These principles are too context-dependent for numeric scoring. When violated, add a prose finding to
the findings list (tagged `[DESIGN]`) rather than a metrics table row. Do not add extra rows to the
canonical table.

---

## Where to place the table in each skill's output

| Skill | Placement |
|-------|-----------|
| `ygs-pr-risk` | After "Key Risks" section, before `findings.json` |
| `ygs-code-review` | After Step 7 (Proportionality check), before Step 8 (Fix-first) |
| `ygs-review-deep` | Phase 3: after dimension-count table, before ranked findings |
| `ygs-pr-audit` | In Metrics Dashboard section (rows added to existing dashboard) |
| `ygs-codebase-audit` | In Metrics Dashboard section (rows added to existing dashboard) |
