---
name: ygs-enrich-ticket
description: >
  Use when a ticket has ACs but no implementation plan, before sprint planning to
  ground tickets in codebase reality, or when ygs-wbs cannot find a PRD. Grids each
  ticket against the codebase and writes a scoped implementation plan back. Supports
  single-ticket and batch mode.
argument-hint: "[<ticket-url-or-id> | --sprint <name> | --label <label>] [--dry-run] [--force] [--format json]"
---

# Enrich Ticket

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.

An implementation plan written before the work starts costs 30 minutes. An implementation plan inferred mid-sprint from incomplete context costs a sprint.

## When NOT to use

- ACs are still missing or vague — run `/ygs-ac-writer` first; enriching a ticket without ACs generates a plan for the wrong thing
- The ticket is blocked or `wontfix` — skip enrichment until unblocked
- The codebase hasn't been cloned or built — the codebase analysis step will produce noise

## Phase 1: Manager — fetch and confirm

Using queries from `~/.claude/skills/you-got-skills/skills/shared/tracker.md`, fetch the ticket list:
- Single ticket: fetch by URL or ID
- `--sprint <name>`: fetch all open tickets in the sprint
- `--label <label>`: fetch all open tickets with the label

Show the list. Confirm with the user before dispatching workers.

Check for existing enrichment markers (`<!-- ai-plan:start -->` / `<!-- ai-plan:end -->`). Skip already-enriched tickets unless `--force` is set.

If `--dry-run`: show which tickets would be enriched and stop. Report **DONE**.

## Phase 2: Worker — one Task subagent per ticket

Dispatch a self-contained worker subagent per ticket using `~/.claude/skills/you-got-skills/skills/shared/subagent-dispatch.md`. The worker prompt includes: ticket content, repo root path, and a reference to the confidence rubric. Workers do not inherit session context.

Each worker runs this pipeline:

### 2.1 Fetch ticket

Read full ticket body, ACs, and comments.

### 2.2 Codebase analysis

```bash
# Find entry points relevant to the ticket
grep -rl "<ticket-keyword>" src/ lib/ --include="*.ts" --include="*.rs" --include="*.go" --include="*.py" 2>/dev/null | head -15
```

Trace the call graph from the identified entry points. Find: affected data models, API layer, storage layer. Note any existing utilities that reduce scope.

### 2.3 Last-touch analysis

```bash
# Who last changed these files and what for?
git log --oneline --follow -n 5 -- <relevant-files>
git blame <relevant-file> | head -30
```

Look for recent changes to the same files — prior work often explains why the current ask wasn't already done, and surfaces hidden constraints.

### 2.4 Skill recommendation

Based on ticket type, note which ygs skills are most relevant for implementation (e.g., schema migrations → `/ygs-deprecate`; API changes → `/ygs-api-review` after implementation).

### 2.5 Score confidence

Apply `~/.claude/skills/you-got-skills/skills/shared/confidence-rubric.md`. Return the score and the evidence basis.

### 2.6 Draft implementation plan

Structure:
```markdown
## Implementation Plan

**Approach:** [1–2 sentence summary]

**Files to change:**
- `path/to/file` — what changes and why

**Dependencies / risks:**
- [known dependency or risk]

**Suggested skills:** [which ygs skills to invoke during implementation]
```

## Phase 3: Manager — write back

Based on confidence score from `~/.claude/skills/you-got-skills/skills/shared/confidence-rubric.md`:

**HIGH (≥70):** Write implementation plan to ticket description using idempotent markers:
```
<!-- ai-plan:start -->
## Implementation Plan
...
<!-- ai-plan:end -->
```

**MEDIUM (40–69):** Write plan with caveat header (see rubric for exact text).

**LOW (<40):** Do not write. Post a comment flagging what's missing and why confidence is low.

If `--format json`: emit `{"tickets":[{"id":"...","status":"written|skipped|low-confidence","score":N}]}` and stop.

## Completion

Report **DONE** with:
- N tickets enriched (HIGH/MEDIUM), M flagged (LOW), K skipped (already enriched)
- Any tickets where the codebase analysis found significant unknowns

Suggest: `/ygs-sprint-ready` to re-validate sprint readiness, `/ygs-implement` to start work.
