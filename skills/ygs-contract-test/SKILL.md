---
name: ygs-contract-test
description: API contract testing — record interactions, derive contracts, detect breaking changes, validate producer/consumer compatibility. Uses api-mock-service or manual recording.
argument-hint: "<service-or-pr> [--mode record|validate|diff] [--base-contract <path>] [--format openapi|pact]"
---

# Contract Test

For test execution, read `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.

Use when you need to verify API compatibility between services — catching breaking changes before they reach production. For fuzz testing (finding crashes/vulnerabilities via mutations), use `/ygs-fuzz-test`.

## Step 1: Determine contract testing mode

| Mode | When to use | What happens |
|------|-------------|--------------|
| `record` | First time, or updating baseline contracts | Run tests through proxy, capture interactions, generate contracts |
| `validate` | PR review, CI gate | Compare current behavior against existing contracts |
| `diff` | Breaking change detection | Compare two contract versions, report differences |

Default to `validate` if a baseline contract exists in the repo, otherwise `record`.

## Step 2: Record API interactions

### With api-mock-service (preferred)

If `api-mock-service` is available (check with `which api-mock-service` or look for it in the project's docker-compose):

```bash
# Start in recording mode
api-mock-service --mode record \
  --proxy-port 8081 \
  --target http://localhost:8080 \
  --data-dir ./recordings

# Run tests through the proxy
HTTP_PROXY=http://localhost:8081 pytest tests/integration/
# or
HTTP_PROXY=http://localhost:8081 npm test -- --testPathPattern=integration
```

### Without api-mock-service (manual recording)

Instrument test HTTP calls to capture request/response pairs:

```python
# Python: use responses or httpretty to record
import responses
import json

recorded = []

@responses.activate(passthrough_prefixes=("http://localhost",), record=True)
def test_api():
    # ... test code ...
    pass

# After tests, extract recorded calls
for call in responses.calls:
    recorded.append({
        "method": call.request.method,
        "path": call.request.path_url,
        "request_headers": dict(call.request.headers),
        "request_body": call.request.body,
        "status_code": call.response.status_code,
        "response_body": call.response.text
    })
```

For other languages, use the language's HTTP mocking library in passthrough+record mode.

## Step 3: Generate contracts from recordings

### OpenAPI contract derivation

From recorded interactions, derive the OpenAPI schema:

1. Group recordings by `method + path`
2. For each endpoint:
   - Infer request schema from request bodies (merge multiple samples)
   - Infer response schema from response bodies per status code
   - Extract path parameters from URL patterns
   - Note required vs optional fields (present in all samples vs some)

```bash
# With api-mock-service
api-mock-service --mode contract \
  --data-dir ./recordings \
  --output ./contracts/openapi.yaml
```

### Pact contract format

If using consumer-driven contracts (Pact style):

1. Consumer tests define expectations
2. Interactions are captured as Pact JSON
3. Provider verifies against Pact contracts

```json
{
  "consumer": {"name": "frontend"},
  "provider": {"name": "user-service"},
  "interactions": [
    {
      "description": "get user by ID",
      "request": {"method": "GET", "path": "/users/123"},
      "response": {
        "status": 200,
        "body": {"id": 123, "name": "string", "email": "string"}
      }
    }
  ]
}
```

## Step 4: Validate / detect breaking changes

Compare the generated contract against the baseline:

### Breaking change categories

| Category | Severity | Example |
|----------|----------|---------|
| **Removed endpoint** | Critical | `DELETE /users/:id` no longer exists |
| **Removed field** | Critical | Response no longer includes `email` |
| **Type change** | Critical | `id` changed from `integer` to `string` |
| **New required field** | High | Request now requires `tenant_id` |
| **Status code change** | High | Was 200, now 201 |
| **Narrowed enum** | Medium | Removed allowed value from enum |
| **New optional field** | Safe | Response added `created_at` |
| **Widened enum** | Safe | Added new allowed value |

### Validation rules

For each endpoint in the baseline contract:
1. Verify it still exists in the new contract
2. For request schemas: new contract must accept everything the old one accepted (no new required fields, no narrowed types)
3. For response schemas: new contract must return everything the old one returned (no removed fields, no type changes)
4. Additional fields in responses are always safe (additive changes)

```bash
# With oasdiff (if available)
oasdiff breaking ./contracts/baseline.yaml ./contracts/current.yaml

# With api-mock-service
api-mock-service --mode contract \
  --baseline ./contracts/baseline.yaml \
  --current ./contracts/current.yaml \
  --report-breaking
```

## Step 5: Report

- **Mode:** record | validate | diff
- **Endpoints covered:** N endpoints, M interactions recorded
- **Contract format:** OpenAPI 3.x | Pact v4
- **Breaking changes:** N critical, N high, N medium, N safe
- **Breaking change details:** for each breaking change:
  - Endpoint affected
  - Change type (removed field, type change, etc.)
  - Before / After
  - Consumer impact assessment
- **Coverage gaps:** endpoints in baseline not exercised by tests

Report **DONE** if no breaking changes detected (or mode is `record`).
Report **DONE_WITH_CONCERNS** if:
- Breaking changes found (list each with severity)
- Coverage gaps exist (endpoints not tested)
- Contract derivation was uncertain (ambiguous types from limited samples)

Suggest: `/ygs-fuzz-test` to stress-test the contract boundaries with mutations.
Suggest: `/ygs-api-review` for broader API design review.
