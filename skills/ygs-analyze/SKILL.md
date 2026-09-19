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

### Phase 0: Ground Truth (MANDATORY — every claim must trace to evidence gathered here)

**Rule: No evidence → no claim. If you didn't read it from the supplied text or a file, you cannot assert it.**

**Two sub-modes — choose based on whether a cloned repo is available:**
- **Repo present** (`## Git Repository (Cloned — Read Files Directly)` is in the input) → run Steps 0a–0f
- **No repo** → run Step 0a then Step 0g (comment mining), then skip to Phase 1

#### 0g. Comment mining (no-repo mode only)

When no cloned repo is provided, the issue's comments, linked tickets, and attachments are the ground truth. Read all comments in chronological order and extract:

1. **Design decisions** — what approach was chosen and why (e.g., "team agreed on SSE over polling")
2. **Rejections** — what was proposed and ruled out (e.g., "polling ruled out due to UX lag")
3. **Blocker status** — for each blocker ticket referenced, what does the issue text say about its status? Note: if a linked issue shows `[Closed]` in the issue data, the blocker may be resolved
4. **Last blocking comment** — who wrote it, when, and what specifically is blocked
5. **Open questions** — anything explicitly marked as unresolved or needing a follow-up
6. **File/function references in comments** — extract any code symbols mentioned; they often point to exactly where work is needed

Build a decision timeline: `[date] person — decision/event` ordered oldest→newest. This replaces the evidence table when there is no repo.

#### 0a. Extract all clues from the issue text

Read **everything supplied**: title, description, comments, attachments, linked issue summaries, and any tables or lists. Pull out:

1. **File paths named explicitly** (e.g., `src/sluice/js/sse/ServerSentEventEmitter.ts`) → open these with `Read` or `cat` immediately; do not grep for them
2. **Class / function names** (e.g., `publishServerEvents`, `useSSE`) → grep for these
3. **Feature keywords** (e.g., `SSE`, `HTTP/2`, `worker_connected`) → grep for these
4. **Blocker ticket IDs** (e.g., `CRIBL-19038`) → note for Step 0d
5. **Reference patterns mentioned in description** (e.g., "similar to `Captures.ts`") → read the referenced file, but also note any explicit caveats in the issue ("but does not spawn a separate process" = different mechanism, not an exact template)
6. **Any additional context from comments** — comments often contain workarounds, test cases, reproduction steps, or file references that sharpen the analysis

#### 0b. Open explicitly-named files

For every file path from 0a, open and read the full file. Then check for:

```bash
# Open and read each file directly — do NOT skip this step
cat <repo_path>/src/path/to/File.ts
# or use the Read tool with the absolute path
```

For each file record:
- **Purpose and key function names** with line numbers
- **TODOs** — copy the exact text; they identify known gaps
- **Wired / partial / stub / dead**
- **Error handling**: scan every line that writes to a network object — `response.write()`, `stream.send()`, `socket.emit()`, `res.write()`. Is it inside a `try/catch` (or equivalent)? If NOT, note "silent failure risk: uncaught write error will propagate and abort delivery to remaining clients in the same iteration." A TODO comment about retries does NOT substitute for this check — explicitly state whether try/catch exists.
- **Hardcoded limits**: scan for numeric literals like `> 2`, `=== 6`, `max = 100` that could be capacity ceilings
- **Access modifiers**: `protected` (intended for subclassing) vs `private` (sealed) — note if `protected` with no subclass
- **Keep-alive / ping**: is there a periodic heartbeat sent to the client? If not and the endpoint is a long-lived stream, note "proxy idle-timeout risk"
- **Deduplication**: in registration/subscription methods, is the same ID checked before adding? If not, note "duplicate registration risk"

#### 0c. Grep for remaining keywords AND check git history

```bash
# 1. Keyword grep across source (backend)
grep -r "ServerSentEvent\|publishServerEvents\|useSSE" <repo_path>/src -l 2>/dev/null | head -30
find <repo_path>/src -name "*SSE*" -o -name "*ServerSentEvent*" 2>/dev/null | head -20
grep -r "<keyword>" <repo_path>/src -l 2>/dev/null | head -20

# 2. UI / frontend hooks that gate the feature — often the critical missing piece
#    If the feature involves a UI component, grep for hooks, feature flags, or client-side gates:
grep -r "use<FeatureName>\|isSupported\|flags\.allows\|feature/" <repo_path>/src/ui -l 2>/dev/null | head -20
#    Example: if issue is about SSE:
grep -r "useSSE\|isSSESupported\|apiProtocol" <repo_path>/src -l 2>/dev/null | head -20
#    Always read the UI hook file if found — it may reveal gating conditions (e.g. requires HTTP/2 flag)

# 3. Files touched in commits that mention the issue key — often finds files the keyword grep misses
ISSUE_KEY="<e.g. CRIBL-16249>"
git -C <repo_path> log --all --grep="$ISSUE_KEY" --name-only --pretty=format: 2>/dev/null \
  | grep -v "^$" | sort -u | head -20
```

Open and read every file returned by any step. Document: `<path>:<line> — <purpose> — wired/partial/stub/dead`

**To assert something doesn't exist**: show the grep command and its empty output. Never write "no X exists" without that proof.

#### 0d. Verify blockers

For each blocked-by or blocking ticket ID from 0a:

1. **Find its status in the linked issues data** — look for the `[Closed]` or `[Open]` tag appended to the linked issue line in the Issue Context section.
2. **If the blocker shows `[Closed]`**: write explicitly — *"Blocker [ID] appears **CLOSED** — verify whether resolved or abandoned before treating as a hard dependency. This ticket may no longer be blocked."*
   - Do NOT continue to describe a `[Closed]` ticket as an active blocker in Phase 2, Phase 4, or Phase 5.
   - Do NOT write recommendations like "unblock PLAT-4625 to enable X" if PLAT-4625 shows `[Closed]`.
3. **If the status is absent or unknown**: write *"Blocker [ID] scope not in provided data — verify before treating as hard dependency."*
4. **If tickets this issue BLOCKS are `[Closed]`**: note they are already closed; do not list them as pending unblocking work.

Do NOT infer scope or status from a ticket number alone.

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
