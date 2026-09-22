---
name: ygs-fuzz-test
description: API fuzz testing — apply 7 field-level and 4 sequence-level mutation strategies against API endpoints, shrink failures with delta debugging, export JUnit results.
argument-hint: "<service-url-or-contract> [--iterations N] [--strategies all|field|sequence] [--timeout Ns]"
---

# Fuzz Test

For contract testing (recording, validation), read `~/.claude/skills/you-got-skills/skills/ygs-contract-test/SKILL.md`.
For test execution, read `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.

Use when you want to find crashes, unhandled errors, and security vulnerabilities by sending mutated inputs to API endpoints. For contract compatibility testing, use `/ygs-contract-test`.

## Step 1: Gather seed corpus

Before fuzzing, you need valid request samples to mutate. Obtain them from:

| Source | How |
|--------|-----|
| Contract/OpenAPI spec | Parse schema, generate valid examples |
| Recorded interactions | Use recordings from `/ygs-contract-test` record mode |
| Test fixtures | Extract request bodies from integration tests |
| Manual | Ask user for example requests |

For each endpoint, collect at least one valid request that produces a 2xx response. This is the seed corpus.

```bash
# From api-mock-service recordings
ls recordings/**/*.json

# From OpenAPI spec (generate examples)
# Parse the spec and create valid payloads per endpoint
```

## Step 2: Apply mutation strategies

### Field-level mutations (7 strategies)

Apply these to individual fields in the request body:

| # | Strategy | What it does | Example |
|---|----------|-------------|---------|
| 1 | **Missing fields** | Remove a required field | `{"name": "x"}` -> `{}` |
| 2 | **Boundary values** | Replace numbers/strings with boundary values | `42` -> `0`, `-1`, `2147483647`, `9223372036854775807` |
| 3 | **Malformed data** | Swap types (string/number, object/array) | `"hello"` -> `42`, `42` -> `"not_a_number"` |
| 4 | **Null injection** | Set fields to null | `"email": "a@b.com"` -> `"email": null` |
| 5 | **Format violations** | Break format constraints (email, date, URL) | `"a@b.com"` -> `"invalid@@format..com"` |
| 6 | **Security payloads** | Inject SQL, XSS, SSTI, path traversal, JNDI | `"name"` -> `"' OR 1=1 --"` |
| 7 | **Combinatorial nulls** | Null multiple fields simultaneously | `{"a": null, "b": null, "c": "valid"}` |

### Sequence-level mutations (4 strategies)

Apply these to the order and combination of API calls:

| # | Strategy | What it does |
|---|----------|-------------|
| 1 | **Reorder** | Swap the order of dependent calls (create before auth, delete before create) |
| 2 | **Replay** | Send the same request twice (test idempotency) |
| 3 | **Skip steps** | Omit setup steps (access resource without creating it first) |
| 4 | **Interleave** | Mix calls from different sessions/users (test isolation) |

## Step 3: Execute fuzz campaign

For each endpoint in the seed corpus:

1. Select a random strategy (or cycle through all if `--strategies all`)
2. Apply the mutation to a random seed request
3. Send the mutated request to the service
4. Record the response status code and body
5. Classify the result:

| Response | Classification |
|----------|---------------|
| 2xx on invalid input | **Finding: insufficient validation** |
| 500+ | **Finding: unhandled error** (crash) |
| 4xx with details | Expected (good error handling) |
| 4xx generic | Note (could improve error messages) |
| Timeout | **Finding: potential DoS** |
| Connection reset | **Finding: crash** |

```bash
# With api-mock-service
api-mock-service --mode fuzz \
  --target <service-url> \
  --data-dir ./recordings \
  --iterations <N> \
  --output ./fuzz_results.json

# Without api-mock-service: use the mutation functions inline (see Step 2)
```

Continue until `--iterations` reached or `--timeout` expires.

## Step 4: Shrink failures with delta debugging

For each finding (5xx or crash), minimize the mutation to find the smallest reproducing case:

### Delta debugging algorithm

1. Start with the mutated request that triggered the failure
2. Try removing each mutation one at a time
3. If the failure still reproduces with fewer mutations, keep the smaller version
4. Repeat until no single mutation can be removed without fixing the failure

```
Original: {"name": null, "age": -1, "email": "invalid"}  -> 500
Remove name=null: {"age": -1, "email": "invalid"}  -> 500 (still fails)
Remove age=-1: {"name": null, "email": "invalid"}  -> 400 (doesn't fail)
Remove email: {"name": null, "age": -1}  -> 500 (still fails)
Minimal: {"age": -1, "name": null}  -> 500
```

The minimal case tells you exactly which combination of invalid inputs triggers the bug.

## Step 5: Export results

### JUnit XML format (for CI integration)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="fuzz-tests" tests="100" failures="3" errors="1">
  <testcase name="fuzz-001-null_injection" classname="POST /users">
    <failure message="null:email" type="critical">
      Mutated field 'email' to null.
      Response: 500 Internal Server Error
      Body: {"error": "NullPointerException"}
      Minimal reproduction: {"name": "test", "email": null}
    </failure>
  </testcase>
</testsuite>
```

Write to `./fuzz_results.xml` for CI systems to pick up.

### JSON findings format

```json
{
  "iterations": 100,
  "findings": [
    {
      "endpoint": "POST /users",
      "strategy": "null_injection",
      "field": "email",
      "severity": "critical",
      "status_code": 500,
      "minimal_reproduction": {"name": "test", "email": null},
      "response_excerpt": "NullPointerException"
    }
  ],
  "coverage": {
    "endpoints_tested": 5,
    "strategies_applied": 11,
    "total_requests": 100
  }
}
```

## Step 6: Report

- **Campaign summary:** N iterations, N endpoints fuzzed, N strategies applied
- **Findings:** N critical, N high, N medium
- **For each finding:**
  - Endpoint and HTTP method
  - Mutation strategy that triggered it
  - Response status and error excerpt
  - Minimal reproduction (after delta debugging)
  - Severity classification
  - Suggested fix
- **Coverage:** endpoints tested vs total, strategies applied
- **Timing:** total fuzz duration, average request latency

Report **DONE** if no critical or high findings.
Report **DONE_WITH_CONCERNS** if:
- Critical findings (500s, crashes) — list each with minimal reproduction
- Security findings (injection payloads accepted) — flag for immediate review
- Coverage gaps (endpoints not fuzzed due to missing seeds)

Suggest: `/ygs-contract-test` to establish baseline contracts before fuzzing.
Suggest: `/ygs-security-review` for deeper security analysis of findings.
