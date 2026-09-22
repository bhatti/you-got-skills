---
name: ygs-enrich-ticket
description: >
  Use when a ticket is missing acceptance criteria, an implementation plan, or
  both — before sprint planning, before assigning work, or when ygs-wbs finds
  no PRD. Writes whatever is missing: ACs first, then a codebase-grounded
  implementation plan. Supports single-ticket and batch mode.
argument-hint: "[<ticket-url-or-id> | --sprint <name> | --label <label>] [--dry-run] [--force] [--format json]"
---

# Enrich Ticket

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.

A ticket enriched before sprint planning costs 30 minutes. The same ticket enriched mid-sprint — when ACs are missing and an agent guesses at scope — costs a sprint.

## When NOT to use

- The ticket is `blocked` or `wontfix` — skip until unblocked
- The codebase hasn't been cloned — codebase analysis will produce noise

## Step 1: Fetch and assess

Using `~/.claude/skills/you-got-skills/skills/shared/tracker.md`:
- Single ticket: fetch by URL or ID
- `--sprint <name>`: all open tickets in the sprint
- `--label <label>`: all open tickets with the label

For each ticket, produce the gap table from `~/.claude/skills/you-got-skills/skills/shared/ac-format.md`:

| Field | Present? | Quality | Notes |
|-------|----------|---------|-------|
| Why statement | | | |
| Story points | | | |
| Existing ACs | | | |
| DoD specified | | | |
| Out-of-scope block | | | |
| Impl plan present | | | |

Show the table. If `--dry-run`: report **DONE** here — no writes.

Check for existing enrichment markers (`<!-- ai-plan:start -->` / `<!-- ai-plan:end -->`). Skip already-enriched tickets unless `--force`.

## Step 2: Write ACs (only if missing or vague)

If ACs are already clear and testable: skip this step.

**Research:** grep the codebase for types/functions from the ticket title and body. Read linked design docs via `~/.claude/skills/you-got-skills/skills/shared/docs-discovery.md`.

**Generate:** ≥3 given/when/then ACs, ≥1 edge/failure case, out-of-scope block, DoD checklist (per `~/.claude/skills/you-got-skills/skills/shared/ac-format.md`). Estimate story points if absent.

**Write back** after confirmation:
```bash
# GitHub
CURRENT=$(gh issue view <number> --json body --jq '.body')
gh issue edit <number> --body "$CURRENT

## Acceptance Criteria
<generated ACs>"

# JIRA
acli -p jira jira issue edit <KEY> --description "<updated description>"
```

Replace existing `## Acceptance Criteria` section if present; otherwise append.

## Step 3: Write implementation plan (one worker subagent per ticket)

Dispatch a self-contained worker per ticket via `~/.claude/skills/you-got-skills/skills/shared/subagent-dispatch.md`. Worker prompt: ticket content + repo root + confidence rubric reference. No session context inherited.

Each worker:

**3.1 Codebase analysis**
```bash
grep -rl "<ticket-keyword>" src/ lib/ --include="*.ts" --include="*.rs" --include="*.go" --include="*.py" 2>/dev/null | head -15
```
Trace call graph from entry points to data layer. Note existing utilities that reduce scope.

**3.2 Last-touch analysis**
```bash
git log --oneline --follow -n 5 -- <relevant-files>
git blame <relevant-file> | head -20
```
Prior changes reveal hidden constraints and why this wasn't done before.

**3.3 Score confidence** — apply `~/.claude/skills/you-got-skills/skills/shared/confidence-rubric.md`.

**3.4 Draft implementation plan**
```markdown
## Implementation Plan

**Approach:** [1–2 sentence summary]

**Files to change:**
- `path/to/file` — what changes and why

**Dependencies / risks:**
- [known dependency or risk]

**Suggested skills:** [which ygs skills to invoke]
```

## Step 4: Write impl plan back

Per confidence score from `~/.claude/skills/you-got-skills/skills/shared/confidence-rubric.md`:

**HIGH (≥70):** Write inside idempotent markers:
```
<!-- ai-plan:start -->
## Implementation Plan
...
<!-- ai-plan:end -->
```

**MEDIUM (40–69):** Write with caveat header (see rubric).

**LOW (<40):** Do not write. Post a comment explaining what's missing.

If `--format json`: emit `{"tickets":[{"id":"...","status":"written|skipped|low-confidence","score":0}]}` and stop.

## Completion

Report **DONE** with:
- Tickets enriched (ACs written, impl plan written, or both)
- Tickets skipped (already complete) or flagged (LOW confidence)
- Any significant unknowns found during codebase analysis

Suggest: `/ygs-sprint-ready` to validate readiness, `/ygs-implement` to start work.
