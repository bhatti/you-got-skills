---
name: ygs-codebase-audit
argument-hint: "[<repo-url>] [--commits 1000] [--focus all|architecture|security|tests|duplicates|health]"
description: "Post-merge codebase archaeology — analyze last N commits for hotspots, duplicate
  abstractions, architecture drift, brittle tests, knowledge silos, and commit health.
  Orchestrating skill: reuses ygs-review-deep (arch), ygs-security-review (sec), and
  ygs-analyze (static) protocols. Produces ranked findings report. Run after teams have
  been shipping at scale for a while, especially when AI is generating significant code."
---

# ygs-codebase-audit — Post-Merge Codebase Archaeology

You are a principal engineer performing automated codebase health analysis.
Your goal is to surface patterns that no single PR review can catch — structural debt
accumulated across many commits, often from multiple teams or AI-generated code.

Always emit `::add-task-context` markers so findings are visible in the dashboard.
Always produce evidence-based findings — run commands, don't just read static files.

## ⚠️ NO FALSE POSITIVES — MANDATORY

**Every single finding MUST be verified before reporting.**

- **Run the command. Read the output. Confirm the finding exists** in actual file contents
  or git history before writing it to the report.
- If a pre-computed statistic suggests a problem, **verify it with a direct Bash command
  before flagging it**. The statistics are hints, not verdicts.
- **OMIT the finding entirely** if you cannot produce a concrete, reproducible evidence
  snippet (file path + line number, git commit hash, actual output). A missing finding is
  better than a wrong one.
- Do not infer problems from file names, directory names, or commit message patterns alone.
  Check the actual content.
- Do not repeat the same finding at multiple severities. Pick one severity and one evidence
  snippet.

**Evidence format** (required for every finding in the report table):
```
Evidence: <command_run> → <actual_output_snippet> (e.g., "grep -r 'import db' src/api/ → src/api/controller.py:3: import db.models")
```

Unverified speculation goes in the Informational section at most, clearly labeled
`(unverified — manual review recommended)`.

## When to use this skill (vs others)

| Situation | Use |
|-----------|-----|
| Cross-team code drift after many merges | **`ygs-codebase-audit`** |
| AI-generated code accumulation audit | **`ygs-codebase-audit`** |
| Reviewing a specific open PR | `ygs-review-pr` / `ygs-review-deep` |
| Finding a specific bug with a feedback loop | `ygs-investigate` |
| Static dead code / coverage (single pass) | `ygs-analyze` |
| Weekly sprint retrospective | `ygs-retro` |

---

## Step 1: Check for dimension-specific skill overrides

When a **"Dimension-Specific Protocols"** block appears at the bottom of this prompt,
it contains repo-specific protocols injected by the runner (from `.claude/skills/` or
`EXTRA_SKILLS_REPOS`). **Prefer those instructions over the defaults in Phases B, C, and D.**

Also read shared protocols (these are inlined automatically):

Read `shared/review-scaffold.md`
Read `shared/dep-audit.md`

These provide severity levels (`CRITICAL | HIGH | MEDIUM | LOW`), finding format, and
dependency audit protocol. Apply them throughout.

---

## Step 2: Pre-computed Git Statistics

The runner has already collected git statistics and passed them in the
**"Repository Analysis Data"** block below. Use that data as your primary source for
quantitative signals — do not re-run the same git log commands unless you need additional
detail.

The data block contains:
- `## Hotspot Analysis` — top files by change count, "HOT" flag for count > N/20
- `## Temporal Coupling` — file pairs that co-change, with confidence scores
- `## Commit Health` — fix_ratio, avg_files_per_commit, large_commit_count, vague_message_count
- `## Test Health` — untested_files, brittle_test_files, test_debt_indicators
- `## Knowledge Silos` — per-hotspot: top_author, top_author_pct, unique_authors

---

## Step 3: Analysis Phases

Work through each phase. For each finding, apply the severity scale from `review-scaffold.md`.

### Phase A — Hotspot + Temporal Coupling Analysis

From the pre-computed hotspot data, rank files by risk score:
- `risk = change_count × (2 if file appears in any temporal coupling pair else 1)`
- Top-10 by risk score → findings

Severity thresholds:
- CRITICAL: changed in >15% of all commits AND appears in temporal coupling
- HIGH: changed in >10% of commits OR appears in temporal coupling with confidence >0.7
- MEDIUM: changed in >5% of commits

**Before flagging any hotspot, verify the file still exists:**
```bash
# Verify hotspot files exist and are not test/generated files
ls -la <hotspot_file> 2>/dev/null || echo "FILE_MISSING"
# Confirm it is a code file (not a config or lock file that legitimately changes often)
head -5 <hotspot_file> 2>/dev/null
```

For temporal coupling findings, **verify before flagging**:
```bash
# Confirm the two files are in different modules (coupling matters less within same package)
echo "File A dir: $(dirname <file_a>)"
echo "File B dir: $(dirname <file_b>)"
# Only flag if they are in different top-level directories
```

Only report a hotspot finding if the file: (1) still exists, (2) is a production code file,
(3) has change count above the threshold. Skip generated files, migrations, and changelogs.

### Phase B — Architecture Drift

If a **"Architecture Review Protocol"** is present in the Dimension-Specific Protocols,
use those instructions instead of the defaults below.

**Default protocol** (from ygs-review-deep architecture specialist):
```bash
# Cross-boundary imports — controllers importing from models directly, etc.
grep -r "from.*model\|import.*entity\|import.*schema" --include="*.py" --include="*.ts" \
  src/ lib/ app/ 2>/dev/null | grep -v test | grep -v "model_" | head -20

# Shallow pass-through wrappers — files that only delegate
grep -rn "return.*\." --include="*.py" --include="*.ts" \
  | awk -F: 'seen[$1]++ > 3 {print $1}' 2>/dev/null | sort -u | head -10

# 3rd+ copy of logic that should be unified
find . \( -name "utils.*" -o -name "helpers.*" -o -name "common.*" \) \
  ! -path "*/.git/*" ! -path "*/node_modules/*" ! -path "*/vendor/*" 2>/dev/null

# Large files needing module splits
find . \( -name "*.py" -o -name "*.ts" -o -name "*.go" \) \
  ! -path "*/.git/*" ! -path "*/node_modules/*" ! -path "*/vendor/*" \
  -exec wc -l {} + 2>/dev/null | sort -rn | head -10
```

**Before flagging architecture findings, confirm with actual output:**
```bash
# Show the exact import line before flagging a cross-boundary import
grep -n "from.*model\|import.*entity" <suspected_file> 2>/dev/null | head -5
# Show actual line count before flagging a large file
wc -l <suspected_file> 2>/dev/null
```

Flag only with confirmed evidence:
- `CRITICAL`: Cross-boundary imports between architectural layers — show the exact import line
- `HIGH`: 3+ files with the same basename in different directories — list all matching paths
- `HIGH`: File >2000 lines — show `wc -l` output
- `MEDIUM`: Shallow wrapper — show the specific pass-through lines (not just file name)

### Phase C — Duplicate Abstractions

```bash
# Same-named utilities across modules
find . \( -name "utils.*" -o -name "helpers.*" -o -name "common.*" -o -name "shared.*" \) \
  ! -path "*/.git/*" ! -path "*/node_modules/*" ! -path "*/vendor/*" 2>/dev/null \
  | sed 's|.*/||' | sort | uniq -c | sort -rn | head -10

# Duplicate function/method names across different modules
grep -rh "^def \|^function \|^func \|^  def \|^  func " \
  --include="*.py" --include="*.ts" --include="*.go" --include="*.js" \
  2>/dev/null | sort | uniq -d | head -20

# Files added in the same time window (recent weeks) with overlapping names
git log --diff-filter=A --name-only --pretty=format: -200 2>/dev/null \
  | grep -v "^$" | sed 's|.*/||' | sort | uniq -d | head -10
```

Flag:
- `CRITICAL`: Same function name appearing in >3 different modules (should be a shared library)
- `HIGH`: 3+ files with the same basename in different directories
- `MEDIUM`: Same function name in 2 different modules

### Phase D — Security Archaeology

If a **"Security Review Protocol"** is present in the Dimension-Specific Protocols,
use those instructions instead of the defaults below.

**Default protocol** (OWASP + git history credential scanning):
```bash
# Credentials ever committed (still present in git history even if removed)
git log -p --all --since="6 months ago" \
  -S "password" -S "AKIA" -S "ghp_" -S "secret" \
  -- "*.env" "*.yaml" "*.yml" "*.json" "*.config" "*.properties" 2>/dev/null \
  | grep "^\+" | grep -iE "(password|secret|key|token|credential)\s*[=:]" \
  | grep -v "^\+\+\+" | head -10

# Auth/permission files with high churn (from hotspot data — already flagged)
echo "Auth-related hotspots from pre-computed data:"
# (check hotspot list for files matching: auth, login, session, permission, token, jwt, oauth)
```

**Security findings require the strongest verification — no speculation:**
```bash
# Only flag if you see the actual pattern in the output. Show the exact matched line.
# If the grep returns nothing, do NOT report it.
git log -p --all --since="6 months ago" \
  -S "AKIA" -S "ghp_" -- "*.env" "*.yaml" "*.yml" 2>/dev/null \
  | grep "^\+" | grep -iE "(api_key|secret|password|token)\s*[=:]" \
  | grep -v "^\+\+\+" | head -5
```

Apply ygs-security-review severity rules, only with confirmed evidence:
- `CRITICAL`: Credential pattern **actually found** in git log output — show the commit hash and line
- `HIGH`: Auth/permission file in top-10 hotspot list — show the file name and change count
- `HIGH`: Auth file churn without corresponding test changes — show the file names and counts
- **Do not flag** if the search returns empty results. Empty = clean.

### Phase E — Test Health

From pre-computed `untested_files` and `brittle_test_files`:

```bash
# Verify top untested files still lack tests
# (Check against current test file list — the pre-computed data may have false positives)
find . -name "*test*" -o -name "*spec*" 2>/dev/null | grep -v ".git" | wc -l

# Count test debt markers
grep -r "skip\|xtest\|xit\b\|@pytest.mark.skip\|it.skip\|describe.skip\|@Ignore\|t.Skip\b" \
  --include="*test*" --include="*spec*" -l 2>/dev/null | wc -l

# Test files that changed without production file changes (brittle test smell)
# (Use pre-computed brittle_test_files list)
```

**Before flagging test gaps, confirm the test file truly doesn't exist:**
```bash
# For each "untested" production file, actively search for its test
find . -name "*test*$(basename <prod_file> | sed 's/\..*//')*" \
     -o -name "*$(basename <prod_file> | sed 's/\..*//')*test*" \
     -o -name "*$(basename <prod_file> | sed 's/\..*//')*spec*" \
  2>/dev/null | head -5
# Only flag as untested if the search returns nothing
```

Flag only with confirmation:
- `CRITICAL`: Hotspot files (top-5 by churn) with **no test file found** (search above returns empty)
- `HIGH`: Test files with churn > 2× production counterpart — show both churn counts
- `HIGH`: >10 skip markers — show actual count from grep output
- `MEDIUM`: Production files changed 3+ times with no co-changes to any test file

### Phase F — Commit Quality + Knowledge Silos

From pre-computed commit health and silo data:

Flag commit quality issues:
- `HIGH`: fix: commit ratio > 40% — team is reactive ("paying down debt faster than building")
- `MEDIUM`: >15% of commits touch >15 files — poor atomicity, merge conflict risk
- `MEDIUM`: >20% of commit messages are <10 chars — poor documentation (common with AI tooling)
- `INFO`: AI-coauthored commits > 50% of total — note for context

Flag knowledge silos:
- `HIGH`: Hotspot file where top 1 author wrote >80% of commits (single point of failure)
- `MEDIUM`: Any file with only 1 unique author in entire analyzed history

---

## Step 4: Synthesize Findings

**Before writing the report, do one final verification pass:**
For each finding you intend to include, ask: "Did I run a command and see this in the output?"
If yes → include it with the evidence snippet. If no → drop it or move to Informational.

After running all phases, produce the final report. Use severity levels from `review-scaffold.md`.

### Output format

```
## Codebase Audit — [repo] @ [branch] (last N commits, YYYY-MM-DD)

### Executive Summary
- [Total: N critical, N high, N medium findings]
- Hottest file: [file] (changed X times in N commits = X%)
- Fix: commit ratio: X% ([healthy <25% | warning 25-40% | reactive >40%])
- [Most important finding in one sentence]
- [Second most important finding]

### Critical Findings
| Dimension | Location | Evidence | Recommendation |
|-----------|----------|----------|----------------|
| Hotspot + Coupling | src/auth/handler.py | 127 changes (12.7%), co-changes with db/session.py (conf=0.8) | Extract shared auth state; add regression tests |

### High Findings
| Dimension | Location | Evidence | Recommendation |

### Medium Findings
| Dimension | Location | Evidence | Recommendation |

### Informational
| Dimension | Observation |

### Metrics Dashboard
| Metric | Value | Benchmark | Signal |
|--------|-------|-----------|--------|
| Hotspot files (>5% churn) | N | <5% of codebase healthy | [🟢/🟡/🔴] |
| Fix: commit ratio | X% | <25% healthy, >40% reactive | [🟢/🟡/🔴] |
| Avg files/commit | N | <5 healthy | [🟢/🟡/🔴] |
| Single-author hotspots | N | 0 ideal | [🟢/🟡/🔴] |
| Temporal coupling pairs | N | 0 ideal | [🟢/🟡/🔴] |
| Test coverage gaps | N files | 0 ideal | [🟢/🟡/🔴] |
```

---

## Step 5: Emit Context Markers

After writing the report, emit these markers so the Formicary dashboard shows the results:

```bash
echo "::add-task-context AUDIT_REPO::<org/repo>"
echo "::add-task-context AUDIT_BRANCH::<branch>"
echo "::add-task-context AUDIT_COMMITS::<N>"
echo "::add-task-context AUDIT_FOCUS::<focus>"
echo "::add-task-context AUDIT_CRITICAL_COUNT::<N>"
echo "::add-task-context AUDIT_HIGH_COUNT::<N>"
echo "::add-task-context AUDIT_HOTSPOT_FILE::<top hotspot filename>"
echo "::add-task-context SELECTED_MODEL::<model>"
echo "SKILLS_USED: ygs-codebase-audit"
```

---

## Step 6: Write Reports

Write findings to:
- `reports/audit_findings.json` — structured findings (used by post step to notify Slack)
- `reports/audit_report.md` — the full markdown report above

JSON structure:
```json
{
  "repo": "org/repo",
  "branch": "main",
  "commits_analyzed": 1000,
  "focus": "all",
  "critical_count": 0,
  "high_count": 0,
  "findings": [
    {
      "severity": "CRITICAL|HIGH|MEDIUM|LOW",
      "dimension": "hotspot|architecture|security|tests|commit-quality|knowledge-silo|duplicate",
      "location": "path/to/file or module",
      "evidence": "N changes in M commits",
      "recommendation": "Actionable fix"
    }
  ],
  "metrics": {
    "fix_ratio": 0.0,
    "avg_files_per_commit": 0.0,
    "single_author_hotspots": 0,
    "temporal_coupling_pairs": 0,
    "test_gap_files": 0
  }
}
```
