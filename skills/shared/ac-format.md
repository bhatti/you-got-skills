# Acceptance Criteria Format

Shared by: `ygs-ac-writer`, `ygs-sprint-ready`, `ygs-triage`

## AC Template

```
Given <precondition — system state before the action>
When  <action — what the actor does>
Then  <observable outcome — what the system does in response>
```

Every AC must be verifiable by a test or a deterministic manual step. "It works" is not an AC.

## Minimums per ticket

- At least 3 ACs (happy path, alternate path, failure/edge case)
- At least 1 AC covering a failure, boundary, or error path
- At least 1 AC per EARS pattern referenced in the ticket body (When / While / If-Then)

## DoD checklist fields

| Field | Description |
|-------|-------------|
| Unit tests | New behavior covered by unit tests |
| Integration tests | Cross-boundary behavior covered (DB, API, service calls) |
| Documentation | Public API or user-facing behavior documented |
| Observability | New failure paths have structured log entries |
| Backwards compatibility | No breaking change, or migration path documented |

## Out-of-scope block

Every ticket should declare what is explicitly excluded. Format:

```
**Out of scope:**
- [specific thing excluded and why]
```

If nothing is excluded, write: `**Out of scope:** Nothing explicitly excluded.`

## Gap analysis table schema

Use this table to evaluate the current state of a ticket before generating ACs:

| Field | Present? | Quality | Notes |
|-------|----------|---------|-------|
| Why statement | ✅ / ❌ | Clear / Vague / Missing | |
| Story points | ✅ / ❌ | — | |
| Existing ACs | ✅ / ❌ | Testable / Vague / None | |
| DoD specified | ✅ / ❌ | Complete / Partial / Missing | |
| Out-of-scope block | ✅ / ❌ | — | |
| Env/deployment scope | ✅ / ❌ | — | Relevant for multi-env deploys |

Quality ratings:
- **Clear / Testable / Complete** — no action needed
- **Vague / Partial** — can be improved, flag but do not block
- **Missing / None** — must be added before ticket is ready-for-agent
