# API testing with curl and jq

Use `helpers.sh` from ai-dev-tools (`scripts/qa/helpers.sh`; in the Formicary pod it is at `/app/scripts/qa/helpers.sh`, elsewhere set `QA_HELPERS` to its path). Without it, write plain curl + jq and record the same fields by hand. It needs `bash`, `curl` and `jq`; without them the checks are recorded as blocked infrastructure.

## Skeleton

```bash
#!/usr/bin/env bash
# qa-out/api-tests.sh: run by env_manager (QA_BASE_URL, QA_OUT, QA_ALLOW_MUTATIONS are set)
source "${QA_HELPERS:-/app/scripts/qa/helpers.sh}"

qa_case C2                                   # case ids come from plan.json
api_call GET /cart/items
check_http 200 "list returns 200"            # a 401/403 here with no credentials is recorded as blocked, not failed
check_json '.items | length' 0 "cart starts empty"

qa_case C1                                   # mutating: blocked unless --allow-mutations
for i in $(seq 1 10); do api_call POST /cart/items -H 'Content-Type: application/json' -d '{"sku":"A"}'; done
api_call POST /cart/items -H 'Content-Type: application/json' -d '{"sku":"A"}'
check_http 409 "11th item is rejected"
check_json '.code' CART_LIMIT "error code is CART_LIMIT"

qa_summary
```

## Rules the helpers enforce

- **One case id per behavior,** matching `plan.json`. A case's outcome is the worst of its checks; a real failure is never turned into infrastructure by a later blocked check.
- `api_call` sends only to `QA_BASE_URL` (an absolute URL must start with it; anything else is refused) and only GET, HEAD and OPTIONS unless `QA_ALLOW_MUTATIONS=1`. Curl options that change the method, follow redirects, use a proxy or config file, or write files are refused.
- No response, a curl error (a truncated or reset transfer included), or 401/403 with no credentials configured is `blocked` with `infra: true`. It is not a failed test.
- Authorization cases (no credentials, wrong role): once credentials are configured `api_call` always sends them. Override per call with `-H 'Authorization:'` (curl drops the header) or use a second token for another role.
- CLI and UNIT cases have no helper of their own: run the command in `api-tests.sh` and record it, e.g. `qa_case C7; out=$(mytool --flag 2>&1); rc=$?; check_eq 0 "$rc" "exits 0"` (or `qa_pass`/`qa_fail`). A planned case with no record is reported `not-run`.
- Auth: `QA_AUTH_HEADER="Authorization: Bearer ..."` or `QA_USERNAME`/`QA_PASSWORD`. Never default or guessed credentials. Do not call a login endpoint through `api_call` (its response body would be saved as evidence): fetch the token with `curl` directly, outside the helpers, and export it as `QA_AUTH_HEADER`.
- Every check records the command and the first 300 characters of the response; the evidence file keeps the first 4000 bytes. Raw `qa-out/` files are not redacted; the report copies and redacts them into `reports/evidence/`.
- There is no `set -e`: every check runs and `qa_summary` always prints counts and warns when nothing executed. It always exits 0: the report is the gate.
- Assert on the contract (status, error code, field values), not on volatile fields (timestamps, generated ids). Capture an id with `jq -r` and reuse it.
- Poll asynchronous results with a deadline loop, never a fixed delay (`~/.claude/skills/you-got-skills/skills/shared/condition-based-waiting.md`).

## Checks

| Helper | Meaning |
|--------|---------|
| `check_http CODE [msg]` | last status equals CODE |
| `check_json 'JQ' EXPECTED msg` | `jq -r` of the body equals EXPECTED |
| `check_contains TEXT msg` | body contains TEXT |
| `check_eq EXPECTED ACTUAL msg` | arbitrary comparison |
| `qa_pass/qa_fail/qa_skip/qa_blocked msg` | explicit outcomes |

## Cases that catch real bugs

- **Negative control:** the nearest request that must succeed (10th item) next to the one that must fail (11th).
- **Authorization:** the same request with no credentials, wrong role, and another user's resource id.
- **Idempotency:** repeat a create/update and check for duplicates.
- **Error shape:** invalid payload returns a 4xx with the documented error body, not a 500 or an HTML page.
- **No partial writes:** after a rejected request, read the resource back and confirm it is unchanged.
- **Logs:** the wrapper collects `logs/env.log`; regression mode compares error patterns against the baseline.

## Performance (`--perf`, regression mode only)

There is no timing helper: write a small loop yourself. Time a fixed, small number of sequential requests per endpoint (after a warm-up; never enough to load a shared target), record p50 as `{"/path": p50_ms}` in `perf.json` per side, and pass both to `scripts.qa.delta --perf-baseline/--perf-candidate`. A regression is p50 more than 20% slower. Treat single-run timings as noise.
