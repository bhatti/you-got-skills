# Acceptance Criteria Format

Shared by: `ygs-enrich-ticket`, `ygs-sprint-ready`. Grading and statuses: `ticket-readiness.md`.

## AC Template

```
Given <precondition — system state before the action>
When  <action — what the actor does>
Then  <observable outcome — what the system does in response>
```

Every AC must be verifiable by a test or a deterministic manual step. "It works" is not an AC.

Add a `Test:` line to each AC: the test file (the existing one to extend, or the new file's path) and the command, or the exact steps ("open Settings, click Save"). Take the file layout and command from the repository's own testing guide (`AGENTS.md`, `docs/testing.md`, the build file); do not invent a layout, and do not point new tests at a legacy pattern the guide retires. Push edge cases down to unit tests; integration tests only for user- or system-facing flows.

`Then` states what a user or caller observes (UI text, HTTP status and body, CLI output, state a user can read back), never internal state. Do not make a log line the `Then` unless the log line is the product (an audit event, for example): tests assert return values, outcomes and persisted state, not what the code logged. `Given` names a real context ("a project with 20 items"), not "the system is running".

The AC describes the public contract, so it keeps holding through a refactor that preserves that contract. Write for a cold reader (engineer, tester or an agent): no "as discussed", no "investigate and decide" (decide first and write the decision in), and do not restate rules the implementer already reads in the repository's contributor guide.

## Minimums per ticket

- At least 3 ACs (happy path, alternate path, failure/edge case)
- At least 1 AC covering a failure, boundary, or error path
- At least 1 AC per EARS pattern referenced in the ticket body (When / While / If-Then)
- If the ticket changes configuration, a schema or traffic between versions of two components: an AC for an older peer and one for an existing configuration after the upgrade. If the ticket does not say, make it a clarifying question rather than assuming compatibility

## DoD checklist fields

| Field | Description |
|-------|-------------|
| Unit tests | New behavior covered by unit tests |
| Integration tests | Cross-boundary behavior covered (DB, API, service calls) |
| Documentation | Public API or user-facing behavior documented |
| Observability | New failure paths have structured log entries |
| Backwards compatibility | No breaking change, or migration path documented |

## Out-of-scope block

Every ticket should declare what is explicitly excluded, including neighbouring code that looks related and must not be touched. Format:

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
