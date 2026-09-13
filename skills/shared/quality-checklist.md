# Quality Checklist — Master Reference

Canonical quality rules applied across **implement → review → audit → learn**. Skills apply these at different depths — implement/review/self-review go deep per function; audit sweeps at scale across commits/PRs.

**Referenced by:** `ygs-implement`, `ygs-code-review`, `ygs-review-deep`, `ygs-review-pr`, `ygs-pr-audit`, `ygs-codebase-audit`, `ygs-learn`.  
**Do not duplicate rules here in individual skills** — reference this file instead.

---

## Depth Guide

| Skill | Depth | How to apply |
|-------|-------|-------------|
| `ygs-implement` self-review | **Deep** | Check each item per function/module changed; block completion on MUST violations |
| `ygs-code-review` / `ygs-review-deep` | **Deep** | Every item verified against actual code with evidence; report findings by severity |
| `ygs-review-pr` | **Medium** | Focus on changed code only; flag clear violations; skip items with no diff evidence |
| `ygs-learn` Phase 0 | **Light** | Grep-visible signals only; flag patterns obvious without tool invocation |
| `ygs-pr-audit` | **Aggregate** | Patterns across PRs, not per-function; compute rates and trends |
| `ygs-codebase-audit` | **Aggregate** | Specialist tools (lizard, jscpd, grep) on whole codebase; compute metrics |

---

## 1. Sloppiness (Verbosity, Mass, Erosion)

**Reference:** `~/.claude/skills/you-got-skills/skills/shared/sloppiness-metrics.md` — formulas, benchmarks, anti-pattern catalogue, tooling.

Summary thresholds:
- Verbosity ratio > 0.30 = HIGH (AI avg: 0.33; good repos: 0.15)
- Erosion score > 0.55 = HIGH (AI avg: 0.68; good repos: 0.31)
- Verbosity anti-patterns: trivial delegators, wrapper-of-wrapper, duplicated guards, shallow pass-throughs

At **deep** depth: compute with lizard/jscpd; report exact values vs. benchmarks.  
At **aggregate** depth: spot-check top-10 hotspot files; flag obvious patterns.  
At **light** depth: flag only grep-visible patterns (2+ anti-patterns in same diff).

---

## 2. Cyclomatic Complexity (CC)

**Reference:** `~/.claude/skills/you-got-skills/skills/shared/sloppiness-metrics.md#cyclomatic-complexity-cc-thresholds`

Count: `if / else if / for / while / case / catch / && / ||` — each adds 1, start at 1.

| CC | Severity (deep/medium) | Audit signal |
|----|----------------------|--------------|
| > 15 | **MUST** split | Flag any CC > 15 function |
| > 10 | **SHOULD** split | Count functions with CC > 10 |
| 6–10 | Monitor — each branch needs a test | Informational |

At **deep** depth: count branches manually for any function > 30 lines in the diff.  
At **aggregate** depth: run `lizard . --csv` and compute erosion score.

---

## 3. Cyclic Dependencies

**Rule:** No circular imports between modules. Dependencies must flow in one direction (inward). Resolve with interfaces/dependency inversion — never with workarounds like re-export tricks or conditional imports.

**Detection commands:**

```bash
# Python — detect circular imports
pip show pipdeptree 2>/dev/null && python3 -c "
import importlib, sys
# List all circular imports using stdlib's modulefinder
" || pip install pydeps 2>/dev/null

# Simpler: grep-based cycle detection (finds back-edges)
# If module A imports B and B imports A:
for f in $(find . -name "*.py" ! -path "*/.git/*" ! -path "*/__pycache__/*" 2>/dev/null | head -50); do
  module=$(basename $f .py)
  imports=$(grep "^from\|^import" $f 2>/dev/null | sed 's/from //; s/ import.*//' | head -20)
  echo "$module: $imports"
done | awk 'NR==FNR{a[$1]=$2; next} {for(m in a) if(index($0,m) && index(a[m],$1)) print "CYCLE:", $1, "<->", m}' - - 2>/dev/null | head -10

# Go — no stdlib cycle detection; use go build error output
go build ./... 2>&1 | grep "import cycle"

# JavaScript/TypeScript — madge
npx madge --circular --extensions ts,js src/ 2>/dev/null | head -20

# General grep heuristic: A imports B AND B imports A
grep -rn "^import\|^from\|^require" --include="*.py" --include="*.ts" --include="*.js" \
  2>/dev/null ! -path "*node_modules*" | awk -F: '{print $1, $3}' | head -100
```

Severity:
- Circular dependency confirmed → **CRITICAL** (never merge; resolve with abstraction layer)
- Dependency flowing wrong direction (domain → infra) → **MUST**
- Near-cycle (A→B→C→A through 3+ hops) → **HIGH**

---

## 4. Modular Boundaries

**Rule:** Dependencies flow inward only: `infra/adapters → service/application → domain/core`. No layer skipping. No business logic in infrastructure, SDK wrappers, or HTTP handlers.

**Detection:**
```bash
# Find cross-boundary imports (adjust layer names to repo structure)
# Domain importing from infra — wrong direction
grep -rn "from.*infra\|import.*infra\|from.*db\|import.*db\|from.*storage" \
  --include="*.py" --include="*.go" --include="*.ts" \
  $(find . -type d -name "domain" -o -name "core" -o -name "model" 2>/dev/null | head -5) \
  2>/dev/null | grep -v "test\|spec\|mock" | head -20

# Business logic in handler/controller layer
grep -rn "if.*balance\|if.*permission\|if.*subscription\|calculate\|compute" \
  --include="*.py" --include="*.go" --include="*.ts" \
  $(find . -type d -name "handler" -o -name "controller" -o -name "routes" 2>/dev/null | head -5) \
  2>/dev/null | grep -v "test\|spec" | head -20
```

Severity:
- Business logic in handler/infra layer → **MUST**
- Domain importing from infra → **MUST**
- Service layer importing from HTTP/API layer → **HIGH**
- Shared utility with no clear layer ownership → **MEDIUM**

---

## 5. Test Coverage

**Reference:** `~/.claude/skills/you-got-skills/skills/shared/testing-discipline.md` — full testing rules.

Summary:
- ≥ 90% line coverage gate; every new code path covered
- No flaky tests (timing, shared mutable state, fixed ports)
- No mocking internal code — only at system boundaries
- Red-green-refactor — tests written before implementation

At **deep** depth: trace each new function in the diff to its test; flag untested paths.  
At **aggregate** depth: count production files with no test counterpart; compute skip-marker total.

---

## 6. Observability

**Rules (all levels):**
- Every new failure path has a structured log entry (not `print`/`fmt.Println`)
- Log levels: ERROR = page-worthy; WARN = unexpected but recoverable; INFO = significant state transition; DEBUG = diagnostic (gated)
- No PII, secrets, or tokens in any log level
- Every new latency-sensitive operation has a duration metric/histogram
- Correlation IDs propagated across async/service boundaries
- New code paths covered by existing alerts, or new alert warranted

**Detection:**
```bash
# New functions with no logging in hotspot files
grep -n "def \|func \|function " <hotspot_file> 2>/dev/null | while read line; do
  # check if function body contains any log call
  echo "$line"
done

# PII patterns in log calls
grep -rn "log\.\|logger\.\|logging\." --include="*.py" --include="*.go" --include="*.ts" \
  2>/dev/null | grep -i "password\|token\|secret\|ssn\|email\|phone\|credit" | head -10

# Debug logs not gated
grep -rn "log\.Debug\|logger\.debug\|console\.log\|fmt\.Println" \
  --include="*.go" --include="*.py" --include="*.ts" \
  2>/dev/null ! -path "*/test*" ! -path "*/spec*" | head -10
```

At **deep** depth: check every new function for logging discipline.  
At **light** depth: only flag ungated debug logs and obvious PII patterns.

---

## 7. Security Basics (lightweight — for non-security reviews)

For deep security analysis, use `ygs-security-review`. This lightweight checklist applies at all levels.

- No hardcoded secrets, tokens, API keys, or credentials in source
- Input validated at every trust boundary (user input, external API responses)
- Auth/authz checks present on every new endpoint or resource access
- No SQL string interpolation (use parameterized queries)
- No command injection (no `os.system(user_input)`, `eval`, `exec` with external data)
- No sensitive data logged or returned in error responses

```bash
# Hardcoded secrets heuristic
grep -rn "api_key\s*=\s*['\"][A-Za-z0-9]\|password\s*=\s*['\"][^'\"]\|secret\s*=\s*['\"]" \
  --include="*.py" --include="*.go" --include="*.ts" --include="*.js" \
  2>/dev/null ! -path "*/test*" ! -path "*/example*" | grep -v "os\.environ\|os\.Getenv\|process\.env\|config\." | head -10

# SQL injection heuristic
grep -rn "execute(.*%\|execute(.*+\|query(.*%\|query(.*+" \
  --include="*.py" --include="*.go" 2>/dev/null | head -10
```

At **deep** depth: verify each finding against actual code.  
At **light** depth: run grep only; flag confirmed patterns.

---

## 8. SRE Basics (lightweight — for non-SRE reviews)

For deep SRE analysis, use `ygs-sre-review`. This lightweight checklist applies at all levels.

- Every external HTTP/gRPC/DB call has an explicit timeout
- Retry logic uses exponential backoff with jitter (not fixed delay, not infinite retry)
- No unbounded queues or caches (always set max size)
- Graceful shutdown handles in-flight requests (drains on SIGTERM)
- No hard startup dependencies (service starts and degrades if dependencies unavailable)

```bash
# HTTP calls without timeout
grep -rn "requests\.get\|requests\.post\|http\.Get\|axios\.\|fetch(" \
  --include="*.py" --include="*.go" --include="*.ts" 2>/dev/null \
  ! -path "*/test*" | grep -v "timeout" | head -10

# Infinite retry patterns
grep -rn "while True\|for.*retry\|retry_count" \
  --include="*.py" --include="*.go" 2>/dev/null ! -path "*/test*" | head -10
```

At **deep** depth: trace every external call to verify timeout and retry.  
At **light** depth: grep-confirm top 2 patterns only.

---

## Checklist Application by Skill

### ygs-implement self-review (after tests pass, before marking done)
```
[ ] No CC > 10 in new functions (count branches)
[ ] No cyclic imports introduced (check with go build / import graph)  
[ ] Dependencies flow correct direction (domain ← service ← infra)
[ ] Every new failure path has structured logging
[ ] No hardcoded secrets or tokens
[ ] External calls have timeouts
[ ] sloppiness: no trivial delegators or wrapper-of-wrappers introduced
```

### ygs-code-review / ygs-review-deep (per diff)
Apply all 8 items at **deep** depth. Every finding requires evidence (file:line + actual code).

### ygs-review-pr (per PR)
Apply items 1–4 at **medium** depth (changed code only), items 5–8 via grep on changed files.

### ygs-pr-audit (across N PRs)
Track per-PR rates: CC violations caught/missed, cyclic dep introductions, observability gaps, security basics missed. Report trends, not per-function findings.

### ygs-codebase-audit (whole codebase)
Items 1–2: run lizard + jscpd (see `sloppiness.md` specialist).  
Item 3: run language-specific cycle detection tools.  
Items 4–8: use specialist files (architecture, sre, security, test-health).
