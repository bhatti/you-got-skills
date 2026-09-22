---
name: ygs-ac-writer
description: >
  Use when a ticket is missing acceptance criteria, when ACs are vague or untestable,
  when sprint planning identifies AC gaps, or when ygs-triage flags a ticket as not
  ready for agent. Invoke with a ticket URL or ID.
argument-hint: "[<ticket-url-or-id>] [--dry-run] [--format json]"
---

# Acceptance Criteria Writer

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.

Requirements you can't verify are requirements you can't ship. This skill writes testable, traceable acceptance criteria so work is unambiguous before it starts.

## When NOT to use

- The ticket already has clear, testable ACs — run `/ygs-sprint-ready` to confirm readiness instead
- The ticket scope is still open and changing — resolve scope first with `/ygs-refine-prd` or a grilling session
- No ticket exists yet — write the ticket first, then call this skill

## Step 1: Fetch the ticket

```bash
# GitHub
gh issue view <number> --json title,body,labels,milestone,assignees

# JIRA (if configured)
acli -p jira jira issue view <KEY> --output json
```

Fallback: if no tracker configured or ticket URL is inaccessible, ask the user to paste the ticket description.

## Step 2: Gap analysis

Produce the gap analysis table from `~/.claude/skills/you-got-skills/skills/shared/ac-format.md`:

| Field | Present? | Quality | Notes |
|-------|----------|---------|-------|
| Why statement | | | |
| Story points | | | |
| Existing ACs | | | |
| DoD specified | | | |
| Out-of-scope block | | | |
| Env/deployment scope | | | |

Show the completed table to the user.

If `--dry-run`: report **DONE** here with the gap table and no further action.

## Step 3: Research

Grep the codebase for types, functions, and modules referenced in the ticket title and body:

```bash
# Adapt the keyword to what the ticket describes
grep -rl "keyword" src/ lib/ --include="*.ts" --include="*.rs" --include="*.go" --include="*.py" 2>/dev/null | head -10
```

Read any linked design docs found via `~/.claude/skills/you-got-skills/skills/shared/docs-discovery.md`. Note the relevant entry points, data shapes, and error paths.

## Step 4: Generate ACs

Using the format from `~/.claude/skills/you-got-skills/skills/shared/ac-format.md`:

- Minimum 3 given/when/then ACs
- Minimum 1 edge case or failure path AC
- Out-of-scope block (explicit)
- DoD checklist (check which fields apply to this ticket)

Map each AC to the EARS pattern it satisfies (When / While / If-Then / The system shall). If a proposed AC doesn't map to any pattern, it's either missing context (go back to Step 3) or gold-plating (remove it).

## Step 5: Estimate story points (if missing)

If story points are absent, estimate using:
- S (1–2 pts): 1–3 files, <300 lines, clear path
- M (3–5 pts): 4–8 files, 300–800 lines, some unknowns
- L (8+ pts): 8+ files or cross-service; suggest splitting

Flag the estimate but do not block the write-back on missing points.

## Step 6: Confirm and write back

Show the generated ACs to the user.

If `--format json`: output the following and stop — do not write to the ticket.

```json
{
  "ticket": "<id>",
  "gap_table": [...],
  "acceptance_criteria": [...],
  "dod": [...],
  "out_of_scope": [...],
  "estimated_points": <number or null>
}
```

Otherwise: confirm with the user before writing. On confirmation, write ACs to the ticket body:

```bash
# GitHub — update issue body (append ACs below existing content)
CURRENT=$(gh issue view <number> --json body --jq '.body')
gh issue edit <number> --body "$CURRENT

## Acceptance Criteria
<generated ACs here>"

# JIRA
acli -p jira jira issue edit <KEY> --description "<full updated description>"
```

If an `## Acceptance Criteria` section already exists in the body: replace it. Otherwise append.

## Step 7: Completion

Report **DONE** with:
- Ticket ID and title
- ACs written (count)
- DoD fields included
- Story points added or flagged
- Any gaps that remain and need human input

Suggest: `/ygs-enrich-ticket <ticket-url>` to add an implementation plan, `/ygs-estimate` if story points need discussion.

---

## Common Rationalizations

| Rationalization | Reality |
|----------------|---------|
| "ACs slow us down — we'll figure it out in implementation" | Implementing against vague requirements produces correct-but-wrong code. The fix is always more expensive than the AC. |
| "The developer knows what to do" | Knowledge in a person's head is not in the ticket. The agent picking this up next session does not share context. |
| "We can add ACs after" | "After" means after the implementation is done — when the ACs become acceptance of what was built, not acceptance of what was needed. |
