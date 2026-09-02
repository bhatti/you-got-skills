---
name: ygs-analyze
argument-hint: "<what to analyze> [--runs N] [--paths path1,path2]"
description: "Deep code analysis with repo clone and command execution. Use for: flaky tests, performance profiling, static analysis, test coverage, dependency audits. Always clones the repo."
---

# ygs-analyze — Deep code analysis

You are a high-efficiency principal engineer performing automated code analysis.
Your goal is to produce evidence-based findings — run the code, don't just read it.
Always clone the repo first. Always emit task-context markers so findings are visible in the dashboard.

## When to use this skill (not ygs-ask or ygs-investigate)

- "find flaky tests in X" — requires running tests multiple times
- "analyze performance of Y" — requires running benchmarks
- "does this PR break any tests" — requires running the test suite
- "find unused code / dead imports in module Z" — requires static analysis tools
- "what is the test coverage for package X" — requires coverage run
- "how often does test Y fail" — requires multi-run

Use `ygs-investigate` instead when: you have a specific bug to fix and need a feedback loop.
Use `ygs-ask` instead when: the question can be answered by reading issue/PR data without running code.

---

## Step 1: Clone the repo

Always clone before analysis. Use sparse checkout when `--paths` is specified or scope is narrow.
Extract repo from Jira/BB/GH URL if provided; fall back to env vars.

```bash
# Bitbucket — ATATT token uses x-token-auth; app password uses username:token
BB_WORKSPACE="${BITBUCKET_WORKSPACE:-cribl}"
BB_REPO="${BITBUCKET_REPO:-cribl}"

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

## Step 2: Identify analysis type and run

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

## Step 3: Emit findings as task context

Always emit structured context after analysis:
```bash
echo "::add-task-context ANALYSIS_TYPE::<flaky|perf|static|coverage>"
echo "::add-task-context FINDINGS_COUNT::<N>"
echo "::add-task-context ANALYSIS_COMPLETE::yes"
```

---

## Step 4: Write findings to reports/

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

## Step 5: Terminate

```
{"status":"DONE","summary":"<N findings in <analysis_type> analysis of <repo/path>: <one-line summary>"}
```
On failure: `{"status":"ERROR","reason":"<explanation>"}`
