# Acceptance Criteria Format

Shared by: `ygs-enrich-ticket`, `ygs-sprint-ready`, `ygs-triage`. Grading and statuses: `ticket-readiness.md`.

## AC Template

```
Given <precondition — system state before the action>
When  <action — what the actor does>
Then  <observable outcome — what the system does in response>
```

Every AC must be verifiable by a test or a deterministic manual step. "It works" is not an AC.

Add a `Test:` line to each AC: the exact command, or the exact steps ("open Settings, click Save"). `Then` states what a user, caller or log shows (UI text, HTTP status and body, a log line), never internal state. `Given` names a real context ("a project with 20 items"), not "the system is running".

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

## Gap analysis

Ticket gaps are graded Pass / Partial / Missing with quoted evidence, and statuses are computed from the grades: `ticket-readiness.md` defines both and `scripts.tickets.readiness` renders the report. Do not keep a second table here.

Rules that stay with the format:
- A Partial AC is converted, not replaced: keep the author's wording and intent.
- A rewrite that adds a `Test:` line and an observable `Then` is the improvement; say which part of the original it came from.
- Missing items must be added before a ticket is ready-for-agent; Partial items are flagged but do not block.
