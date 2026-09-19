---
name: ygs-analyze
argument-hint: "<what to analyze> [--runs N] [--paths path1,path2]"
description: "Deep analysis: issue/bug archaeology (root cause, PR blame, spec/review/test gaps, systemic patterns) AND code analysis (flaky tests, perf, static analysis, coverage). Detects mode from input."
---

# ygs-analyze — Deep analysis

You are a principal engineer performing evidence-based analysis. You have **two modes**:

- **Issue Analysis Mode** — activated when the input contains structured issue/ticket data (Jira or GitHub issues, bug reports, feature specs). Focuses on root cause, archaeology, process gaps, systemic patterns.
- **Code Analysis Mode** — activated when given a direct code analysis task (flaky test, benchmark, coverage). Clones the repo and runs commands.

**Detect mode from context:** If "## Issue Context to Analyze" or "## Issues" is in the input → Issue Analysis Mode. Otherwise → Code Analysis Mode.

---

## ISSUE ANALYSIS MODE

Use this mode when analyzing Jira/GitHub issues (bugs, features, specs) with or without git context.

### Phase 0: Repo Exploration (MANDATORY when cloned repo is available)

**This phase determines the ground truth of the codebase. Git commit recency is NOT a reliable
indicator of implementation state — code may exist from an earlier sprint. You MUST read actual
source files before drawing any conclusions.**

If the prompt contains a `## Git Repository (Cloned — Read Files Directly)` section:

**Step 1 — Extract keywords from the issue** (before looking at the repo):
Read the issue title, description, component labels, and linked issue summaries. Identify 3–6
technical keywords (class names, endpoint paths, service names, feature flags, config keys).
Example for "streaming worker connection events": `ServerSentEvent`, `SSE`, `ConnectionListener`,
`publishServerEvents`, `worker_connected`, `useSSE`.

**Step 2 — Grep the full codebase for each keyword**:
```bash
# Search for each keyword — DO NOT rely solely on recent git history
grep -r "ServerSentEvent\|SSE\|publishServer" <repo_path>/src -l --include="*.ts" --include="*.py" --include="*.go" 2>/dev/null | head -30
grep -r "<keyword2>" <repo_path>/src -l 2>/dev/null | head -20
find <repo_path> -name "*<KeywordPattern>*" 2>/dev/null | head -20
```

**Step 3 — Read every relevant file found**:
For each file returned: use `Read` to open it, understand its purpose, note key
function names, line numbers, TODOs, and whether it is wired up or dead code.
Document: `<path>:<line> — <what it does> — <wired/partial/stub/dead>`

**Step 4 — Check tests**:
```bash
find <repo_path> -name "*.test.*" -o -name "*.spec.*" | xargs grep -l "<keyword>" 2>/dev/null | head -10
```
Note which files have test coverage and which are untested.

**Step 5 — Summarise implementation state BEFORE reading git context**:
List what EXISTS, what is WIRED, what is PARTIAL, and what is MISSING.
This prevents git recency bias from distorting your analysis.

Cite specific file paths and line numbers throughout your analysis (e.g., `src/foo/bar.ts:42`).

### Phase 1: Issue Classification

For each issue, identify:
- **Type**: Bug / Feature / Tech Debt / Spike / Incident
- **Severity** (for bugs): P0 Critical / P1 High / P2 Medium / P3 Low
- **Component/area**: which service, module, or subsystem is affected
- **Age**: how long has this been open? (from `created` field)

Emit:
```bash
echo "::add-task-context ISSUE_TYPE::<Bug|Feature|Tech Debt|...>"
echo "::add-task-context ISSUE_SEVERITY::<P0|P1|P2|P3|N/A>"
```

### Phase 2: Root Cause Analysis (Bugs) / Feature State Analysis

**Ground truth = issue description + Phase 0 codebase grep. Git commit recency is irrelevant to
implementation state. Never conclude "work has not started" from recent git activity alone.**

For **bug issues**, perform a structured root cause analysis using the **5-Why** method:

1. **Symptom**: What exact behavior was observed vs. expected? (from issue description)
2. **Trigger**: What user action / system event triggered it?
3. **Immediate cause**: What line of code / config / data caused the failure? (cite file:line from Phase 0)
4. **Root cause**: Why did that code exist / get merged?
5. **Contributing factors**: What conditions made this worse or harder to detect?

For **feature/blocked issues**, assess actual implementation state using Phase 0 findings:
- What is ALREADY built? (list files:lines found in Phase 0)
- What is WIRED to production vs. behind a flag?
- What is truly missing vs. what exists but is not connected?
- Why is it blocked? (read the blocking issue description if referenced)

**Git context** (`## Git Repository Context`) is useful ONLY for:
- Finding the commit that introduced a specific bug (grep by issue key or keywords)
- Understanding which PRs touched which files
- NOT for determining whether a feature was implemented (use Phase 0 grep instead)

Format:
```
### Root Cause Analysis
**Symptom**: ...
**Immediate cause**: ...
**Root cause**: ...
**Introducing commit**: `{hash}` by {author} on {date}
  PR: {url if available}
**Contributing factors**: ...
```

### Phase 3: Process Gap Analysis

Analyze what systemic gaps allowed this issue to be created or go undetected:

#### 3a. Specification Gaps
- Was the requirement ambiguous or missing edge cases?
- Did the spec/AC cover the failure scenario? (check linked issues, description)
- Were acceptance criteria clear and testable?

#### 3b. Code Review Gaps
- Based on the commit/PR context, what review checks would have caught this?
- Was complexity too high for effective review?
- Were there red flags in the diff that a reviewer might have flagged?

#### 3c. Testing Gaps
- What type of test should have caught this: unit / integration / E2E / contract / chaos?
- Was the component under-tested (check hot files from git context)?
- Was there a test for the happy path but not the failure path?

#### 3d. Observability Gaps
- Could monitoring/alerting have detected this earlier?
- Were there missing logs or metrics that would have surfaced the issue?

Format each gap as:
```
**Spec gap**: <yes/no/partial> — <explanation>
**Review gap**: <yes/no/partial> — <what check was missed>
**Test gap**: <yes/no/partial> — <what test was missing>
**Observability gap**: <yes/no/partial> — <what was not monitored>
```

### Phase 4: Systemic Patterns (Systems Thinking)

Look across ALL issues in this batch and identify:

1. **Recurring patterns**: Do multiple issues share the same root cause type? (e.g., "3 of 5 bugs are missing null checks at API boundaries")
2. **High-risk components**: Which files/modules appear repeatedly in git hot-files or issue descriptions?
3. **Process breakdowns**: Is there a pattern in spec gaps? Review gaps? Test gaps?
4. **Prevention levers**: What systemic changes would prevent this class of bug?
   - Linting rule / static analysis check
   - New test template or coverage requirement
   - Spec checklist addition (e.g., "always include error cases")
   - Review guideline addition (e.g., "check for null on all external inputs")
   - Architectural change (e.g., move validation to a shared layer)

Format:
```
### Systemic Patterns
**Recurring pattern**: ...
**High-risk components**: `file1.py`, `file2.ts`
**Process breakdown**: ...

### Prevention Recommendations
1. **Short-term (this sprint)**: <specific PR/task to address immediate risk>
2. **Medium-term (this quarter)**: <process or test change>
3. **Long-term (architectural)**: <systemic change>
```

### Phase 5: Priority & Effort

For each issue provide:
- **Priority recommendation**: P0/P1/P2/P3 with justification
- **Effort estimate**: XS (< 1 day) / S (1-2 days) / M (3-5 days) / L (1-2 weeks) / XL (> 2 weeks)
- **Suggested assignee type**: frontend / backend / platform / security / data

### Phase 6: Write output

**CRITICAL: Two separate steps. The emit commands must NEVER appear inside the report.md file.**

**Step 6a — Write the analysis to reports/report.md:**

```bash
mkdir -p reports
cat > reports/report.md << 'REPORT_EOF'
## <First section heading>
...full analysis here...
REPORT_EOF
```

Rules for report.md content:
- Use `## Section` headings, `**bold**`, `- ` bullets, `` `code` ``
- Include all issue URLs as `[text](url)` links
- Include specific file paths and line numbers discovered in Phase 0
- Do NOT include a title heading — start directly with the first section
- Do NOT include any echo commands, task-context markers, or signoff lines
- End the file with the last analysis content — nothing after it

The `reports/report.md` content is what gets posted to Slack and rendered as HTML.

**Step 6b — Emit task context (run AFTER closing REPORT_EOF, as separate commands):**

```bash
echo "::add-task-context ANALYSIS_COMPLETE::yes"
echo "::add-task-context BUGS_ANALYZED::<N>"
echo "::add-task-context GAPS_FOUND::<spec|review|test|observability>"
```

These echo commands are NOT part of report.md. Run them in the shell after writing the file.

---

## CODE ANALYSIS MODE

Use this mode for direct code analysis tasks (flaky tests, performance, static analysis, coverage).
Activate when there is no "## Issue Context to Analyze" section in the input.

### When to use this mode (not Issue Analysis Mode)

- "find flaky tests in X" — requires running tests multiple times
- "analyze performance of Y" — requires running benchmarks
- "does this PR break any tests" — requires running the test suite
- "find unused code / dead imports in module Z" — requires static analysis tools
- "what is the test coverage for package X" — requires coverage run

Use `ygs-investigate` instead when: you have a specific bug to fix and need a feedback loop.
Use `ygs-ask` instead when: the question can be answered by reading issue/PR data without running code.

---

### Step C1: Clone the repo

Always clone before analysis. Use sparse checkout when `--paths` is specified or scope is narrow.
Extract repo from Jira/BB/GH URL if provided; fall back to env vars.

```bash
# Bitbucket — ATATT token uses x-token-auth; app password uses username:token
BB_WORKSPACE="${BITBUCKET_WORKSPACE:-def-repo}"
BB_REPO="${BITBUCKET_REPO:-def-repo}"

if [[ "${BITBUCKET_TOKEN}" == ATATT* ]]; then
  CLONE_URL="https://x-token-auth:${BITBUCKET_TOKEN}@bitbucket.org/${BB_WORKSPACE}/${BB_REPO}.git"
else
  CLONE_URL="https://${BITBUCKET_USERNAME}:${BITBUCKET_TOKEN}@bitbucket.org/${BB_WORKSPACE}/${BB_REPO}.git"
fi

# Sparse clone (fast, 2-5s) when paths are known
git clone --depth 1 --filter=blob:none --sparse "$CLONE_URL" /tmp/repo 2>&1 | tail -3
git -C /tmp/repo sparse-checkout set <path1> <path2>
git -C /tmp/repo checkout

# Full clone (slow, use only when scope is broad or paths unknown)
git clone --depth 1 "$CLONE_URL" /tmp/repo 2>&1 | tail -3

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

---

### Step C2: Identify analysis type and run

### A) Flaky test detection

Run the target test(s) N times (default: 10). Collect pass/fail/error per run.
Identify: timing-sensitive, order-dependent, resource-dependent failures.

```bash
cd /tmp/repo
# Install deps if needed (check for package.json, requirements.txt, go.mod, etc.)
# Run target test N times; capture exit code and output per run
RUNS=10
PASS=0; FAIL=0; ERRORS=""
for i in $(seq 1 $RUNS); do
  OUTPUT=$(pytest <test_path> -x --tb=short -q 2>&1) && PASS=$((PASS+1)) || {
    FAIL=$((FAIL+1))
    ERRORS="$ERRORS\n--- Run $i ---\n$(echo "$OUTPUT" | tail -20)"
  }
done
echo "Results: $PASS/$RUNS passed, $FAIL failed"
echo "$ERRORS"
echo "::add-task-context FLAKY_TEST_RUNS::$RUNS"
echo "::add-task-context FLAKY_TEST_PASS_RATE::$PASS/$RUNS"
echo "::add-task-context FLAKY_DETECTED::$([ $FAIL -gt 0 ] && echo yes || echo no)"
```

For order-dependent flakiness, also run with randomized order:
```bash
pytest <test_path> -p randomly --randomly-seed=random -q 2>&1 | tail -20
```

### B) Performance / benchmark analysis

```bash
cd /tmp/repo
# Run benchmark tool appropriate for the language (pytest-benchmark, go test -bench, etc.)
# Capture: mean, p95, p99, allocations
# Compare against baseline if available
```

### C) Static analysis (unused code, complexity, imports)

```bash
cd /tmp/repo
# Python
ruff check <path> --select F401,F811,C901 2>/dev/null   # unused imports, complexity
vulture <path> --min-confidence 80 2>/dev/null           # dead code

# JavaScript/TypeScript
npx ts-prune 2>/dev/null | head -50                      # unused exports
npx depcheck 2>/dev/null | head -30                      # unused dependencies

# Go
go vet ./... 2>/dev/null
staticcheck ./... 2>/dev/null | head -30
```

### D) Test coverage

```bash
cd /tmp/repo
# Python
pytest <path> --cov=<module> --cov-report=term-missing -q 2>&1 | tail -40
echo "::add-task-context COVERAGE_RUN::yes"

# Go
go test -coverprofile=coverage.out ./... && go tool cover -func=coverage.out | tail -20
```

---

### Step C3: Emit findings as task context

Always emit structured context after analysis:
```bash
echo "::add-task-context ANALYSIS_TYPE::<flaky|perf|static|coverage>"
echo "::add-task-context FINDINGS_COUNT::<N>"
echo "::add-task-context ANALYSIS_COMPLETE::yes"
```

---

### Step C4: Write findings to reports/

Always write:
- `reports/analysis.md` — full findings with evidence (test output, file:line refs, run stats)
- `reports/summary.txt` — one paragraph for Slack

Format findings as:
```
## Finding: <name>
Severity: CRITICAL | HIGH | MEDIUM | LOW | INFO
Evidence: <exact test output or tool output>
File: <path>:<line>
Recommendation: <specific, actionable fix>
```

---

### Step C5: Terminate

```
{"status":"DONE","summary":"<N findings in <analysis_type> analysis of <repo/path>: <one-line summary>"}
```
On failure: `{"status":"ERROR","reason":"<explanation>"}`
