# Breaking-Change Signals

Single source for what counts as a backward-incompatible change and which tests prove it safe. Used by `ygs-pr-audit`
(changelog specialist, via `scripts/analyze/breaking_changes.py`), and by `ygs-api-review` and `ygs-review-pr` when they judge
a change to a public contract.

## What breaks consumers
| Kind | Example | Confidence from a diff |
|------|---------|------------------------|
| Removed or renamed contract element | proto field, rpc, message; OpenAPI path, property; GraphQL field | high |
| Changed meaning on the wire | field renumbered or retyped; enum value removed, renumbered or repurposed | high |
| Stricter input | new required property or parameter, narrower type, new validation | high (required) / medium |
| Changed output a caller branches on | status code, error body or code, ordering, pagination, null vs empty | medium |
| Removed or re-signatured public symbol | exported function, class, SDK method, CLI flag | medium / low |
| Destructive or locking data change | drop or rename column/table, `NOT NULL` without default, type change, new unique/foreign key | high / medium |
| **Same contract, different behaviour** | changed default, limit, timeout, retry count, rounding, sort order, auth scope, feature flag default | medium / low: needs judgement |
| Declared by the author | `feat!:`, `BREAKING CHANGE`, `breaking` label, "requires migration" | high |

The last two rows are the ones tests miss: nothing fails to compile, callers just behave differently. A heuristic can only
point at them (a changed value for a key such as `timeout`, `limit`, `default`); a person or model must read the change.

## Blast radius
- **External**: customers, partners, public SDKs/CLIs, published APIs, docs users rely on.
- **Internal**: other teams whose code uses the changed symbol, file or table (found by searching the repository for the
  changed names and mapping the hits to their code owners).
- **Always large**: auth/identity, security/crypto, billing/payments, SRE/infrastructure/deploy. Treat any change there as
  high even when it is small.

## Tests a team can run
| Signal | Test |
|--------|------|
| Proto/OpenAPI/GraphQL contract change | contract diff (`buf breaking`, `oasdiff`, schema compatibility) and consumer-driven contract tests |
| Enum value change or removal | exhaustive-switch tests in each consumer; load persisted data containing the old value |
| Required input added | requests from an old client must still be accepted or fail with a clear error |
| Route, method or status code change | client tests that branch on the old status; access-log check that the old route has no callers |
| Database change | expand/contract rehearsal on production-sized data; old and new app versions against the new schema; rollback |
| Default/limit/timeout/flag change | staged rollout (canary) with metrics; compare behaviour before and after on real traffic |
| Public symbol or signature change | build and run every dependent; search for remaining usages |

Report candidates with evidence and confidence. Never present a heuristic as proof, and never invent consumers.
