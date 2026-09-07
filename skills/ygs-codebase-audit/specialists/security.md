# Specialist: Security Archaeology

Review scope: credentials ever committed to git history, auth file churn, injection vectors in hotspot files, missing auth checks on new endpoints.

**Rule: Only report findings where the command actually returns output. Empty output = clean. No speculation.**

## Step 1: Credentials in git history

These may have been removed from current code but are still in git history — a permanent exposure risk.

```bash
# ATATT tokens, AWS keys, GitHub PATs, generic secrets
git log -p --all --since="12 months ago" \
  -S "AKIA" -S "ghp_" -S "ATATT" -S "sk-" -S "xoxb-" \
  -- "*.env" "*.yaml" "*.yml" "*.json" "*.config" "*.properties" \
  "*.toml" "*.ini" "*.conf" 2>/dev/null \
  | grep "^\+" | grep -v "^\+\+\+" \
  | grep -iE "(api_key|secret|password|token|credential|private_key)\s*[=:\"']" \
  | head -10

# Generic password/secret patterns in all file types (last 6 months)
git log -p --since="6 months ago" \
  -S "password" -S "secret" -S "private_key" 2>/dev/null \
  | grep "^\+" | grep -v "^\+\+\+" \
  | grep -iE "password\s*[=:]\s*['\"][^$\{]" \
  | grep -v "test\|mock\|example\|placeholder\|changeme\|TODO\|FIXME" \
  | head -5
```

If any output: show the exact matched line and note the commit hash (get from surrounding context). Do NOT show the actual secret value — mask it as `***`.

## Step 2: Auth/permission file churn

Cross-reference the hotspot file list with security-sensitive file patterns:

```bash
# Find auth/session/permission files in the hotspot list
# (Run against top-20 hotspot files from pre-computed data)
echo "Checking hotspot files for auth-related patterns..."
for f in <hotspot_file_1> <hotspot_file_2> ...; do
  if echo "$f" | grep -qiE "auth|login|session|permission|token|jwt|oauth|credential|access|role|acl|rbac"; then
    count=$(git log --oneline -- "$f" 2>/dev/null | wc -l)
    echo "AUTH-SENSITIVE HOTSPOT: $f ($count changes)"
  fi
done
```

For each auth-sensitive hotspot: check if corresponding test files changed in tandem:

```bash
git log --oneline -50 -- <auth_file> 2>/dev/null | while read hash msg; do
  test_changes=$(git diff-tree --no-commit-id -r --name-only "$hash" 2>/dev/null | grep -iE "test|spec" | wc -l)
  total=$(git diff-tree --no-commit-id -r --name-only "$hash" 2>/dev/null | wc -l)
  if [ "$test_changes" -eq 0 ]; then echo "no-test: $hash $msg"; fi
done 2>/dev/null | head -5
```

## Step 3: Injection vectors in hotspot files

For the top-10 hotspot files, check for common OWASP vulnerabilities:

```bash
# SQL injection: string concatenation in queries
grep -n "execute\|query\|cursor\." <hotspot_file> 2>/dev/null \
  | grep -v "?" | grep -v "parameterized\|prepared\|bind" \
  | grep "+" | head -5

# Command injection: user input to shell
grep -n "exec\|spawn\|system\|popen\|subprocess\|child_process\|shell=True" \
  <hotspot_file> 2>/dev/null | head -5

# Unsanitized output: XSS vectors
grep -n "innerHTML\|dangerouslySetInnerHTML\|v-html\|\.html(" \
  <hotspot_file> 2>/dev/null | head -5

# SSRF: user-controlled URL fetch
grep -n "fetch\|axios\|requests\.get\|http\.get\|urllib" \
  <hotspot_file> 2>/dev/null | head -5
```

Read surrounding context (5 lines) before flagging — verify user input actually flows to the vulnerable call.

## Step 4: New endpoints without auth middleware

```bash
# Find route/endpoint definitions added in last 200 commits
git log --diff-filter=A -200 --name-only --pretty=format: 2>/dev/null \
  | grep -iE "route|controller|handler|endpoint|api" | grep -v "^$" | head -20

# For each new endpoint file, check for auth middleware
for f in <new_endpoint_files>; do
  if [ -f "$f" ]; then
    has_auth=$(grep -ciE "auth|middleware|authenticate|authorize|require_login|@login_required|requireAuth" "$f")
    echo "$f: auth_refs=$has_auth"
  fi
done
```

## Severity guidance

| Finding | Severity | Confidence |
|---------|----------|-----------|
| Credential pattern found in git history (confirmed output) | CRITICAL | HIGH |
| Auth file in top-5 hotspot with no test co-changes | HIGH | HIGH |
| SQL/command injection vector confirmed by reading call site | CRITICAL | HIGH |
| New endpoint file with zero auth references | HIGH | MEDIUM |
| Injection pattern found but unclear if input reaches it | MEDIUM | MEDIUM |

## Finding format

```
#### [SECURITY] <title> — <file>:<line> | Confidence: HIGH
**Evidence:** `git log -p -S "AKIA" -- *.yml | grep "^\+"` → `+AWS_ACCESS_KEY_ID: AKIA***REDACTED*** (commit a3f2c1b)`
**Impact:** AWS key exposed in git history — rotated or not, the commit hash is permanent.
**Recommendation:** Rotate the key immediately. Use `git filter-repo` to scrub history. Add pre-commit hook for credential scanning.
```
