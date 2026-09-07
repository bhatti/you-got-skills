# Specialist: SRE / Operational Reliability

Review scope: error handling gaps, missing observability, retry/timeout absent, hardcoded configuration, and blast-radius mapping of hotspot modules.

## Step 1: Error handling gaps in hotspot files

For the top-10 hotspot files, check for bare exception catches and swallowed errors:

```bash
# Python: bare except / empty except clauses
grep -n "except:\|except Exception:\|except Exception as e:\s*$\|except.*pass" \
  <hotspot_file> 2>/dev/null | head -10

# TypeScript/JavaScript: empty catch blocks
grep -n "catch\s*(" <hotspot_file> 2>/dev/null | while read line; do
  lineno=$(echo "$line" | cut -d: -f1)
  # Check if the catch body is just a comment or empty
  sed -n "${lineno},$((lineno+3))p" <hotspot_file> 2>/dev/null
done | head -20

# Go: error assigned to _ (silently discarded)
grep -n ",\s*_\s*:=\|_\s*=.*err" <hotspot_file> 2>/dev/null | head -10
```

For each hit: read 5 lines of context to confirm the error is truly swallowed (not just re-raised or logged differently).

```bash
sed -n '$((LINE-1)),$((LINE+4))p' <hotspot_file> 2>/dev/null
```

## Step 2: Missing observability in frequently-changed modules

Modules that change often but have no logging/metrics are a reliability risk — failures are invisible.

```bash
# Check hotspot files for any logging/metrics instrumentation
grep -c "log\.\|logger\.\|logging\.\|metrics\.\|counter\.\|histogram\.\|gauge\.\|trace\.\|span\." \
  <hotspot_file> 2>/dev/null

# Check for structured logging patterns
grep -n "log\.\|logger\.\|console\.error\|fmt\.Print\|print(" \
  <hotspot_file> 2>/dev/null | wc -l

# Look for metrics instrumentation
grep -rn "prometheus\|statsd\|datadog\|opentelemetry\|newrelic\|metrics\." \
  <hotspot_file> 2>/dev/null | head -5
```

Flag a hotspot file as "missing observability" only if: it has >50 lines AND zero logging/metrics calls AND it is not a pure data-type/interface file.

## Step 3: Missing retry/timeout on external calls

```bash
# HTTP clients without timeout
grep -rn "new.*HttpClient\|axios\.\|fetch(\|requests\.\|http\.Get\|http\.Post\|grpc\." \
  --include="*.ts" --include="*.py" --include="*.go" --include="*.js" \
  ! -path "*/node_modules/*" ! -path "*/test*" 2>/dev/null \
  | grep -v "timeout\|Timeout\|withTimeout\|context\.WithTimeout" | head -15

# DB queries without timeout context
grep -rn "\.query(\|\.execute(\|\.find(\|\.findOne(\|\.aggregate(" \
  --include="*.ts" --include="*.py" --include="*.go" \
  ! -path "*/node_modules/*" ! -path "*/test*" 2>/dev/null \
  | grep -v "timeout\|ctx\|context" | head -10

# Retry logic absent — look for retry libraries/patterns
grep -rn "retry\|backoff\|circuit.*breaker\|exponential" \
  --include="*.ts" --include="*.py" --include="*.go" \
  ! -path "*/node_modules/*" 2>/dev/null | wc -l
```

For HTTP/gRPC calls without timeout: read 3 lines of context to confirm no timeout is set.

## Step 4: Hardcoded configuration

```bash
# Hardcoded URLs / endpoints
grep -rn "https\?://[a-z0-9]" \
  --include="*.ts" --include="*.py" --include="*.go" --include="*.js" \
  ! -path "*/node_modules/*" ! -path "*test*" ! -path "*mock*" ! -path "*.md" \
  2>/dev/null | grep -v "localhost\|127\.0\.0\." | head -10

# Hardcoded ports
grep -rn ":[0-9]\{4,5\}[\"')]" \
  --include="*.ts" --include="*.py" --include="*.go" \
  ! -path "*/node_modules/*" ! -path "*test*" 2>/dev/null | head -10

# Magic numbers in business logic (not in tests/constants)
grep -rn "\b[0-9]\{3,\}\b" \
  --include="*.ts" --include="*.py" --include="*.go" \
  ! -path "*/node_modules/*" ! -path "*test*" ! -path "*constant*" ! -path "*config*" \
  2>/dev/null | grep -v "//.*\|#.*\|timeout\|1000\|1024\|3600\|86400" | head -10
```

## Step 5: Blast radius mapping

For the top-5 hotspot files, measure fan-in (how many other modules import them):

```bash
HOTSPOT="<file_basename_without_extension>"
# TypeScript/JavaScript
grep -r "from.*${HOTSPOT}\|require.*${HOTSPOT}" \
  --include="*.ts" --include="*.js" \
  ! -path "*/node_modules/*" ! -path "*/dist/*" 2>/dev/null | wc -l

# Python
grep -r "from.*${HOTSPOT} import\|import.*${HOTSPOT}" \
  --include="*.py" ! -path "*/__pycache__/*" 2>/dev/null | wc -l

# Go
grep -r "\".*/${HOTSPOT}\"" --include="*.go" 2>/dev/null | wc -l
```

High fan-in (>10 importers) on a hotspot file = broad blast radius. Document the count.

## Severity guidance

| Finding | Severity | Confidence |
|---------|----------|-----------|
| Bare except/empty catch in hotspot file (confirmed empty body) | HIGH | HIGH |
| HTTP/gRPC call without any timeout in top-10 hotspot | HIGH | HIGH |
| Hotspot file (>100 lines) with zero logging AND high fan-in (>10) | HIGH | MEDIUM |
| Hardcoded production URL/endpoint in source | HIGH | HIGH |
| Error silently discarded (`_ = err`) in critical path | HIGH | HIGH |
| DB call without timeout context | MEDIUM | HIGH |
| Missing retry on external call | MEDIUM | MEDIUM |
| Hardcoded port or magic number in business logic | MEDIUM | HIGH |

## Finding format

```
#### [SRE] <title> — <file>:<line> | Confidence: HIGH
**Evidence:** `grep -n "except:" src/worker/processor.py` → `src/worker/processor.py:87: except:`
`sed -n '87,91p' src/worker/processor.py` → `except:\n    pass  # TODO: handle this`
**Impact:** Exception silently swallowed — failures in the worker processor produce no log entry, no metric, and no alert. Incidents invisible.
**Recommendation:** At minimum log the exception with context. Add a Prometheus error counter. If the exception is expected, name the exception type explicitly.
```
