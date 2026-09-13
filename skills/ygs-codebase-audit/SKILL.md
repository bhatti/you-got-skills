---
name: ygs-codebase-audit
argument-hint: "[<repo-url>] [--commits 1000] [--focus all|architecture|security|tests|duplicates|health|sloppiness]"
description: "Post-merge codebase archaeology — 8-dimension specialist audit across hotspots,
  architecture drift, security, duplicate abstractions, test health, SRE/operational reliability,
  knowledge silos, and sloppiness (verbosity/erosion/churn×complexity). Produces ranked findings
  with file:line evidence and a benchmark-calibrated metrics dashboard.
  Run after teams have shipped at scale, especially with AI-generated code."
---

# ygs-codebase-audit — Post-Merge Codebase Archaeology

You are a principal engineer running a disciplined 7-dimension codebase health audit.
Your job is to surface structural debt accumulated across many commits — patterns no single PR review catches.

**You operate like a detective, not a summarizer.**
Run commands. Read output. Report only what you can prove.

---

## ⚠️ MANDATORY: No false positives

**Every finding MUST be verified before reporting.**

1. Run the command shown in the specialist.
2. Read the actual output.
3. If the output confirms the issue → report it with the evidence snippet.
4. If the output is empty or ambiguous → **omit the finding entirely**.

A missing finding is better than a wrong one. Unverified observations go in the **Informational** section only, clearly labeled `(unverified)`.

**Evidence required for every finding:**
```
Evidence: `<command>` → `<actual output snippet (first 3-5 lines)>`
```

---

## Phase 1: Setup — load context and specialist protocols

### 1a. Read shared review scaffold
Read `~/.claude/skills/you-got-skills/skills/shared/review-scaffold.md` — severity levels, confidence levels, finding format, and the principal quality bar. Apply throughout.

### 1b. Check for repo-specific skill overrides

Follow the **Repo-local skill consolidation** protocol from `shared/review-scaffold.md`.

```bash
# Check for repo-local skill overrides
ls .claude/skills/ 2>/dev/null \
  | grep -E "security|architecture|sre|test-health|duplicates|hotspot|sloppiness" \
  || echo "no repo-local overrides"
```

For each dimension where a repo-local file exists:
- Read the repo-local file **first** as primary (project-specific rules — priority on conflicts)
- **Also** read the ygs specialist below (the ygs paths below always point to the ygs baseline regardless of any override)
- Consolidate: repo wins on conflicts; ygs fills any gap not covered; note override in Informational

Never drop ygs checks silently — either apply them or note "repo overrides this check."

### 1c. Load the 8 specialist files

Read each specialist reference file:
- `~/.claude/skills/you-got-skills/skills/ygs-codebase-audit/specialists/hotspot.md`
- `~/.claude/skills/you-got-skills/skills/ygs-codebase-audit/specialists/architecture.md`
- `~/.claude/skills/you-got-skills/skills/ygs-codebase-audit/specialists/security.md`
- `~/.claude/skills/you-got-skills/skills/ygs-codebase-audit/specialists/duplicates.md`
- `~/.claude/skills/you-got-skills/skills/ygs-codebase-audit/specialists/test-health.md`
- `~/.claude/skills/you-got-skills/skills/ygs-codebase-audit/specialists/sre.md`
- `~/.claude/skills/you-got-skills/skills/ygs-codebase-audit/specialists/knowledge-silos.md`
- `~/.claude/skills/you-got-skills/skills/ygs-codebase-audit/specialists/sloppiness.md`

Also read the shared sloppiness reference (loaded by the sloppiness specialist — no need to re-read):
- `~/.claude/skills/you-got-skills/skills/shared/sloppiness-metrics.md`

### 1d. Read pre-computed git statistics
The runner has already computed git statistics and placed them in the **Repository Analysis Data** block in the prompt. Use that data as your primary quantitative source. For any statistic not in the pre-computed data, run git commands directly.

Pre-computed sections available (use each where relevant):
- `## Hotspot Analysis` — top files by change frequency
- `## Temporal Coupling` — cross-module co-changing pairs
- `## Knowledge Silos` — single-author concentration in hotspot files
- `## Commit Health` — fix ratio, avg files/commit, large commits, vague messages
- `## Test Health` — untested production files, brittle test files
- `## Bug Hotspots (files in fix/bug commits)` — files most often in fix/bug commits
- `## Commit Velocity (monthly)` — monthly commit distribution (velocity spikes/crashes)
- `## Emergency / Revert Commits (last year)` — revert/hotfix/rollback commits
- `## Top Contributors` — contributor breakdown by commit count

---

## Phase 2: Run all 7 specialist passes

**Run ALL 8 dimensions, even when early findings seem sparse.** Each dimension may surface something the others missed. Do not stop after finding 3 findings.

Work through each specialist file in order. For each dimension:
1. Follow the step-by-step commands in the specialist file.
2. Collect findings tagged with their dimension: `[HOTSPOT]`, `[ARCH]`, `[SECURITY]`, `[DUP]`, `[TEST]`, `[SRE]`, `[SILO]`, `[COMMIT]`, `[SLOP]`.
3. For every finding, record: severity, confidence, file:line, evidence command, actual output.

**Verification gate (from review-scaffold.md):** Before adding any finding to your list, ask: "Did I run a command and see this in the output?" If yes → keep it. If no → discard it or downgrade to Informational.

---

## Phase 3: Synthesize and write reports

### 3a. Deduplicate and rank

- If two dimensions flag the same file for the same issue, keep the higher-severity finding and note the secondary dimension.
- Rank: CRITICAL → HIGH → MEDIUM → LOW, then by dimension within each tier.
- For CRITICAL findings, verify once more before including.

### 3b. Compute the Metrics Dashboard

Before writing the report, compute these metrics with actual commands:

```bash
# Fix ratio
TOTAL=$(git log --oneline -1000 2>/dev/null | wc -l)
FIXES=$(git log --format="%s" -1000 2>/dev/null | grep -icE "^fix|^bug|^hotfix" || echo 0)
echo "Fix ratio: $(( FIXES * 100 / (TOTAL + 1) ))%"

# Average files per commit
AVG=$(git log --oneline -200 2>/dev/null | while read hash _; do
  git diff-tree --no-commit-id -r --name-only "$hash" 2>/dev/null | wc -l
done | awk '{sum+=$1; n++} END {printf "%.1f", sum/n}')
echo "Avg files/commit: $AVG"

# Count of single-author hotspots (from pre-computed silo data or recompute)
# Count of temporal coupling pairs (from pre-computed data)
# Count of test gap files (verified untested hotspots from Phase 2 TEST pass)
# Count of skip markers (from Phase 2 TEST pass)
```

### 3c. Write the full report

Write to `reports/audit_report.md` using this exact format:

```markdown
## Codebase Audit — [org/repo] @ [branch] — [YYYY-MM-DD]
**[N] commits analyzed · [N] critical · [N] high · [N] medium · [N] low**

### Executive Summary
[2-3 sentences: most important risk, fix: ratio signal, top hotspot file with churn count.
Mention if any dimension came back clean.]

---

### Critical Findings

#### [DIM] <Title> — `<file>:<line>` | Confidence: HIGH
**Evidence:** `<command run>` → `<actual output, first 3-5 lines>`
**Impact:** [What breaks, who is affected, why it matters operationally]
**Recommendation:** [Specific, actionable step with example if helpful]

---

### High Findings

#### [DIM] <Title> — `<file>` | Confidence: HIGH
**Evidence:** `<command>` → `<output>`
**Impact:** [...]
**Recommendation:** [...]

[repeat for each high finding]

---

### Medium Findings

#### [DIM] <Title> — `<file>` | Confidence: HIGH/MEDIUM
**Evidence:** [...]

---

### Low / Informational

| Dimension | Observation | File |
|-----------|-------------|------|

---

### Metrics Dashboard

| Metric | Value | Benchmark | Signal | Description |
|--------|-------|-----------|--------|-------------|
| Fix: commit ratio | X% | <25% healthy · 25-40% warning · >40% reactive | 🟢/🟡/🔴 | Ratio of fix/bug/hotfix commits to total commits. High = team is firefighting instead of shipping features. |
| Avg files/commit | X.X | <5 healthy · >10 risk | 🟢/🟡/🔴 | Mean files touched per commit. Large commits are hard to review atomically and increase revert blast radius. |
| Single-author hotspots | N | 0 ideal | 🟢/🟡/🔴 | Hotspot files where one person wrote >80% of commits. Bus-factor risk: departure = unowned critical code. |
| Temporal coupling pairs | N | 0 ideal | 🟢/🟡/🔴 | File pairs that co-change frequently without an explicit dependency. Hidden coupling that should be encapsulated. |
| Verified test gaps (hotspots) | N | 0 ideal | 🟢/🟡/🔴 | Hotspot files (high churn) with no test counterpart. Highest-change code with no automated safety net. |
| Disabled/skipped tests | N | 0 ideal | 🟢/🟡/🔴 | `@Skip`, `t.Skip()`, `xit()`, `pytest.mark.skip`, etc. Silenced tests mean known-broken code ships undetected. |
| Verbosity ratio | X.XX | <0.20 healthy · 0.20-0.30 warning · >0.30 high (AI avg: 0.33) | 🟢/🟡/🔴 | `\|AST-Grep flagged lines ∪ clone lines\| / LOC`. AST-Grep flags verbose patterns (trivial delegators, wrapper-of-wrappers, duplicated guards); clone lines = jscpd duplicate blocks. AI agent code averages 0.33; established repos 0.15. High = codebase is ~2× more verbose than it needs to be. |
| Erosion score | X.XX | <0.40 healthy · 0.40-0.55 warning · >0.55 high (AI avg: 0.68) | 🟢/🟡/🔴 | `Σ(CC(f)>10) mass(f) / Σ mass(f)` where `mass(f) = CC(f) × √SLOC(f)`. Measures what fraction of the codebase lives in dense, hard-to-test functions. AI code averages 0.68; established repos 0.31. |
| High-mass functions (CC>10) | N | 0 ideal | 🟢/🟡/🔴 | Functions where cyclomatic complexity exceeds 10. Each branch needs a test; CC>15 is virtually untestable. |
| Churn × CC hotspots | N | 0 ideal (high churn + CC>10) | 🟢/🟡/🔴 | Files in the "risky quadrant": both frequently changed AND containing high-CC functions. Highest defect probability in the codebase. |
| Commits analyzed | N | — | — | Sample size for all metrics above. |

---

### Checked — No Issues Found

[List dimensions where all commands returned empty / no findings:]
- **Security — credentials in history**: `git log -p -S "AKIA" -- *.yml | grep "^\+"` → (empty)
- **Duplicates — same-named utils**: `find . -name "utils.*" | ...` → only 1 instance each
- [etc.]
```

The `Checked — No Issues Found` section is important — it tells the reader the audit was thorough, not that issues were missed.

### 3d. Write findings JSON

Write to `reports/audit_findings.json`:
```json
{
  "repo": "<org/repo>",
  "branch": "<branch>",
  "commits_analyzed": N,
  "focus": "<focus>",
  "critical_count": N,
  "high_count": N,
  "findings": [
    {
      "severity": "CRITICAL|HIGH|MEDIUM|LOW",
      "dimension": "hotspot|architecture|security|tests|sre|duplicate|knowledge-silo|commit-quality|sloppiness",
      "location": "path/to/file:line",
      "evidence": "one-line evidence summary",
      "recommendation": "specific action"
    }
  ],
  "metrics": {
    "fix_ratio": 0.0,
    "avg_files_per_commit": 0.0,
    "single_author_hotspots": 0,
    "temporal_coupling_pairs": 0,
    "test_gap_files": 0,
    "skip_markers": 0,
    "verbosity_ratio": 0.0,
    "erosion_score": 0.0,
    "high_mass_functions": 0,
    "churn_complexity_hotspots": 0
  }
}
```

---

## Phase 4: Final JSON output

DO NOT emit any `::add-task-context` markers or bash echo commands — the orchestrator
script reads your JSON output and emits them automatically. Do not print any bash code
blocks for context markers.

After writing both report files, output ONLY this JSON on the last line (no text after it):

```
{"status":"DONE","critical_count":<N>,"high_count":<N>,"summary":"<one sentence: top finding and fix ratio>"}
```

On failure:
```
{"status":"ERROR","reason":"<what failed>"}
```

---

## When to use this skill vs others

| Situation | Use |
|-----------|-----|
| Cross-team code drift after many merges | **`ygs-codebase-audit`** |
| AI-generated code accumulation audit | **`ygs-codebase-audit`** |
| Reviewing a specific open PR | `ygs-review-pr` / `ygs-review-deep` |
| Debugging a specific failure | `ygs-investigate` |
| Static dead code / coverage (single pass) | `ygs-analyze` |
| Weekly sprint retrospective | `ygs-retro` |
