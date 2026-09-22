---
name: ygs-sprint-ready
description: >
  Use before sprint planning to validate that every candidate ticket is ready to
  pull in. Evaluates tickets field-by-field, auto-enriches where possible, flags
  what needs human action. Lighter than running a full sprint plan; pair with
  ygs-sprint-plan after.
argument-hint: "[--sprint <name> | --label <label> | <ticket-url>...] [--dry-run] [--format json]"
---

# Sprint Ready

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.

**Principle:** A ticket that enters sprint planning without ACs or a Why statement will consume grooming time during the sprint, not before it. Fix it now — before the sprint starts.

## When NOT to use

- Sprint is already in progress — use `/ygs-triage` to fix individual tickets mid-sprint
- Tickets have no tracker data (no GitHub/JIRA configured) — use `/ygs-enrich-ticket` on each ticket directly

## Step 1: Fetch ticket list

Using `~/.claude/skills/you-got-skills/skills/shared/tracker.md`:

- `--sprint <name>`: all open tickets in the named sprint or sprint backlog
- `--label <label>`: all open tickets with the given label
- `<ticket-url>...`: explicit list of ticket URLs

Show the list. Confirm scope with user before proceeding.

If `--dry-run`: evaluate tickets and show readiness table, but do not call any enrichment skills or write to tickets. Report **DONE** after showing the table.

## Step 2: Evaluate readiness

For each ticket, check 6 fields using `~/.claude/skills/you-got-skills/skills/shared/ac-format.md`:

| Field | Check |
|-------|-------|
| Why statement | Body explains the user or business value — not "we need to do X" |
| Story points | Points are set (any non-zero value) |
| ACs written | At least 1 acceptance criterion in the body |
| ACs testable | ACs use Given/When/Then or a specific observable outcome — not vague phrases like "it works" |
| DoD specified | At least unit tests and backwards compatibility addressed |
| Impl plan present | `<!-- ai-plan:start -->` marker found in ticket body |

Assign a priority-ordered status to each ticket:

```
BLOCKED          — labelled blocked or has an unresolved blocking dependency
NEEDS_AC_WORK    — ACs missing or not testable
NEEDS_STORY_POINTS — no story points set
NEEDS_IMPL_PLAN  — no implementation plan (all other fields OK)
READY            — all 6 fields pass
```

## Step 3: Produce readiness table

```
TICKET    TITLE                      STATUS              GAPS
PROJ-38   Auth refactor              READY               —
PROJ-41   Rate limiter               NEEDS_AC_WORK       ACs missing
PROJ-52   Dashboard metrics          NEEDS_IMPL_PLAN     Impl plan absent
PROJ-60   External API integration   BLOCKED             Blocked by PROJ-38
```

Cross-sprint summary at the bottom:

```
READY: N  |  NEEDS_HUMAN: M  |  AUTO_ENRICHABLE (candidates): K
Most common gaps: NEEDS_AC_WORK (N), NEEDS_IMPL_PLAN (N)
```

`AUTO_ENRICHABLE` = tickets this skill can fix without human input (NEEDS_AC_WORK or NEEDS_IMPL_PLAN status).

## Step 4: Auto-enrich (skip if `--dry-run`)

For `NEEDS_AC_WORK` and `NEEDS_IMPL_PLAN` tickets:
1. Run `/ygs-enrich-ticket --dry-run <ticket-url>` — show the gap table and what would be written
2. Confirm per ticket (or "enrich all" for batch confirmation)
3. Run `/ygs-enrich-ticket <ticket-url>` — writes ACs if missing, then impl plan

For `BLOCKED` and `NEEDS_STORY_POINTS` tickets: flag for human — these cannot be auto-enriched.

## Step 5: Re-evaluate and action items

After enrichment, re-check enriched tickets. Update the readiness table.

Final action items section — three buckets:

```
AUTO-ENRICHED (agent fixed these):
  PROJ-41 — ACs written ✅
  PROJ-52 — Impl plan added ✅ (confidence: HIGH)

NEEDS HUMAN (cannot auto-enrich):
  PROJ-60 — BLOCKED: waiting on PROJ-38 to close
  PROJ-63 — NEEDS_STORY_POINTS: no estimate possible without design decision

READY TO PULL IN (N tickets):
  PROJ-38, PROJ-41, PROJ-52
```

## Completion

If `--format json`:
```json
{
  "tickets": [
    {"id": "PROJ-38", "status": "READY", "gaps": []},
    {"id": "PROJ-41", "status": "NEEDS_AC_WORK", "gaps": ["no_acs"]}
  ],
  "summary": {"ready": 0, "needs_human": 0, "auto_enriched": 0}
}
```
Replace `0` with actual counts.
```

Otherwise, report **DONE** with the readiness table and action items.

If any tickets remain `NEEDS_HUMAN`: report **DONE_WITH_CONCERNS** — list what humans must fix before those tickets can be pulled in.

Suggest: `/ygs-sprint-plan` to proceed with planning once READY count reaches target capacity.
