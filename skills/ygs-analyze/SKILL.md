---
name: ygs-analyze
argument-hint: "<what to analyze> [--runs N] [--paths path1,path2]"
description: "Deep analysis: issue/bug archaeology (root cause, PR blame, spec/review/test gaps, systemic patterns) AND code analysis (flaky tests, perf, static analysis, coverage). Detects mode from input."
---

# ygs-analyze — Deep analysis

**Two modes** (auto-detected from input):
- **Issue Analysis** — input has `## Issue Context to Analyze` → run Phases 0–6 below
- **Code Analysis** — direct code task (flaky test, benchmark, coverage) → skip to Code Analysis Mode

---

## ISSUE ANALYSIS MODE

### Phase 0: Ground Truth (MANDATORY — every claim must trace to a line read here)

**Rule: No file read → no claim. If you didn't read it, you cannot assert it.**

#### 0a. Extract all clues from the issue text

Read **everything supplied**: title, description, comments, attachments, linked issue summaries, and any tables or lists. Pull out:

1. **File paths named explicitly** (e.g., `src/sluice/js/sse/ServerSentEventEmitter.ts`) → open these with `Read` or `cat` immediately; do not grep for them
2. **Class / function names** (e.g., `publishServerEvents`, `useSSE`) → grep for these
3. **Feature keywords** (e.g., `SSE`, `HTTP/2`, `worker_connected`) → grep for these
4. **Blocker ticket IDs** (e.g., `CRIBL-19038`) → note for Step 0d
5. **Any additional context from comments** — comments often contain workarounds, test cases, reproduction steps, or file references that sharpen the analysis

#### 0b. Open explicitly-named files

For every file path from 0a:
```bash
# Open and read each file directly — do NOT skip this step
cat <repo_path>/src/path/to/File.ts
# or use the Read tool with the absolute path
```
Record for each: purpose, key function names, line numbers, TODOs, whether it is wired or stub/dead.

#### 0c. Grep for remaining keywords

```bash
grep -r "ServerSentEvent\|publishServerEvents\|useSSE" <repo_path>/src -l 2>/dev/null | head -30
find <repo_path>/src -name "*SSE*" -o -name "*ServerSentEvent*" 2>/dev/null | head -20
grep -r "<keyword>" <repo_path>/src -l 2>/dev/null | head -20
```

Open and read every file returned. Document: `<path>:<line> — <purpose> — wired/partial/stub/dead`

**To assert something doesn't exist**: show the grep command and its empty output. Never write "no X exists" without that proof.

#### 0d. Verify blockers

For each blocked-by ticket ID from 0a: search the supplied issue text for that ticket's title and description.
If not found: write "Blocker [ID] scope not in provided data — verify before treating as hard dependency."
Do NOT infer scope from a ticket number alone.

#### 0e. Check tests

```bash
find <repo_path> -name "*.test.*" -o -name "*.spec.*" 2>/dev/null | \
  xargs grep -l "<keyword>" 2>/dev/null | head -10
```

#### 0f. Build evidence table (required before Phase 1)

Compile everything found into a table — this gets included in the final report:

| File | Key symbol / line | Status |
|------|------------------|--------|
| `src/foo/Bar.ts:42` | `publishEvents()` | wired \| partial \| stub \| dead |
| *(grep for "X" returned empty)* | — | MISSING |

Only mark MISSING when you have an empty grep result to prove it. If you didn't search, say "not checked."

---

### Phase 1: Issue Classification

- **Type**: Bug / Feature / Tech Debt / Spike / Incident
- **Severity**: P0 Critical / P1 High / P2 Medium / P3 Low / N/A
- **Component**: which service, module, or subsystem

```bash
echo "::add-task-context ISSUE_TYPE::<Bug|Feature|Tech Debt|...>"
echo "::add-task-context ISSUE_SEVERITY::<P0|P1|P2|P3|N/A>"
```

---

### Phase 2: Root Cause / Implementation State

**Every claim must cite a file:line from Phase 0. Use "likely", "possibly", "not verified" when you cannot cite a line.**

**For bugs** — 5-Why:
1. **Symptom**: observed vs. expected (from issue description)
2. **Immediate cause**: `file:line` (must be in Phase 0 evidence table)
3. **Root cause**: why that code existed / was merged
4. **Introducing commit**: grep git log by issue key or keywords

**For features / blocked issues** — use Phase 0 evidence only:
- **EXISTS**: already built — list file:line
- **WIRED**: connected to production vs. behind a flag — cite flag name and file:line
- **PARTIAL**: exists but not connected — show where the wiring is missing
- **MISSING**: only if grep returned empty — cite the grep
- **Blockers**: cite from issue text; if absent, write "blocker scope not verified"

**Git context** (`## Git Repository Context`) is useful only for:
- Finding the commit that introduced a bug (grep by issue key)
- Understanding which PRs touched a file
- **NOT** for determining whether a feature was implemented (use Phase 0 grep)

---

### Phase 3: Process Gaps

For each gap, cite Phase 0 evidence or issue text:
- **Spec gap**: yes/no/partial — what's ambiguous or missing from the spec/AC
- **Review gap**: yes/no/partial — what check in review would have caught it
- **Test gap**: yes/no/partial — missing test type; cite test file (or empty grep proving no tests)
- **Observability gap**: yes/no/partial — what metric/log is absent

---

### Phase 4: Systemic Patterns & Prevention

Across all issues in this batch:
- **Recurring pattern**: what root cause type appears more than once
- **High-risk components**: files that appear repeatedly (from Phase 0 + git hot files)
- **Process breakdown**: which gap type dominates

Prevention (concrete, not generic):
1. **Short-term (this sprint)**: specific PR/task
2. **Medium-term (this quarter)**: process or test change
3. **Long-term**: architectural change that eliminates the class of problem

---

### Phase 5: Priority & Effort

Per issue:
- **Priority**: P0–P3 with justification grounded in Phase 0 (existing infrastructure → lower risk)
- **Effort**: XS (<1d) / S (1–2d) / M (3–5d) / L (1–2w) / XL (>2w) — factor in what already exists
- **Assignee type**: frontend / backend / platform / security / data

---

### Phase 6: Write output

**CRITICAL: Two separate steps. Echo commands must NEVER appear inside report.md.**

**Step 6a — Write to `reports/report.md`:**

```bash
mkdir -p reports
cat > reports/report.md << 'REPORT_EOF'
## <First section heading>
...full analysis here — include Phase 0 evidence table...
REPORT_EOF
```

Rules for `report.md` content:
- `##` headings, `**bold**`, `- ` bullets, `` `code` ``
- All issue URLs as `[text](url)` links
- Include the Phase 0 evidence table
- Start with the first section heading — no title line above it
- No echo commands, task-context markers, or signoff lines anywhere in the file
- End with the last analysis content — nothing after it

**Step 6b — Emit task context (run AFTER closing `REPORT_EOF`, as separate shell commands):**

```bash
echo "::add-task-context ANALYSIS_COMPLETE::yes"
echo "::add-task-context BUGS_ANALYZED::<N>"
echo "::add-task-context GAPS_FOUND::<spec|review|test|observability>"
```

---

## CODE ANALYSIS MODE

Activate when there is **no** `## Issue Context to Analyze` section in the input.

Use for: flaky test detection, performance benchmarks, static analysis, test coverage.

Use `ygs-investigate` instead when you need a fix-and-verify feedback loop.
Use `ygs-ask` instead when the question can be answered from issue/PR data without running code.

### C1: Clone the repo

```bash
# Bitbucket (ATATT token uses x-token-auth; app password uses username:token)
if [[ "${BITBUCKET_TOKEN}" == ATATT* ]]; then
  CLONE_URL="https://x-token-auth:${BITBUCKET_TOKEN}@bitbucket.org/${BITBUCKET_WORKSPACE}/${BITBUCKET_REPO}.git"
else
  CLONE_URL="https://${BITBUCKET_USERNAME}:${BITBUCKET_TOKEN}@bitbucket.org/${BITBUCKET_WORKSPACE}/${BITBUCKET_REPO}.git"
fi
git clone --depth 1 --filter=blob:none --sparse "$CLONE_URL" /tmp/repo 2>&1 | tail -3

# GitHub
CLONE_URL="https://x-access-token:${GH_TOKEN}@github.com/${GH_ORG}/${GH_REPO}.git"
git clone --depth 1 --filter=blob:none --sparse "$CLONE_URL" /tmp/repo 2>&1 | tail -3
```

After cloning:
```bash
echo "::add-task-context REPO_CLONED::yes"
echo "::add-task-context REPO_CLONED_PATH::/tmp/repo"
echo "::add-task-context CLONE_TYPE::<sparse|full>"
```

### C2: Run analysis

**Flaky test detection** — run N times, collect pass/fail:
```bash
RUNS=10; PASS=0; FAIL=0; ERRORS=""
for i in $(seq 1 $RUNS); do
  OUTPUT=$(pytest <test_path> -x --tb=short -q 2>&1) && PASS=$((PASS+1)) || {
    FAIL=$((FAIL+1)); ERRORS="$ERRORS\n--- Run $i ---\n$(echo "$OUTPUT" | tail -20)"
  }
done
echo "Results: $PASS/$RUNS passed"
echo "$ERRORS"
echo "::add-task-context FLAKY_DETECTED::$([ $FAIL -gt 0 ] && echo yes || echo no)"
```

**Static analysis:**
```bash
ruff check <path> --select F401,F811,C901 2>/dev/null        # Python: unused imports, complexity
npx ts-prune 2>/dev/null | head -50                          # TS: unused exports
go vet ./... && staticcheck ./... 2>/dev/null | head -30     # Go
```

**Test coverage:**
```bash
pytest <path> --cov=<module> --cov-report=term-missing -q 2>&1 | tail -40
```

### C3: Emit context

```bash
echo "::add-task-context ANALYSIS_TYPE::<flaky|perf|static|coverage>"
echo "::add-task-context FINDINGS_COUNT::<N>"
echo "::add-task-context ANALYSIS_COMPLETE::yes"
```

### C4: Write findings

Write `reports/analysis.md` — full findings with evidence (file:line, test output, run stats).

Format each finding:
```
## Finding: <name>
Severity: CRITICAL | HIGH | MEDIUM | LOW | INFO
Evidence: <exact output or file:line>
Recommendation: <specific, actionable fix>
```

### C5: Terminate

```
{"status":"DONE","summary":"<N findings in <analysis_type> of <repo>: <one-line summary>"}
```
On failure: `{"status":"ERROR","reason":"<explanation>"}`
