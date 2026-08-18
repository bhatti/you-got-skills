---
name: ygs-handoff
description: Compress the current session into a structured handoff doc for the next session — open tasks, key decisions, context gaps, and suggested next actions.
argument-hint: "[--output <path>]"
---

# Handoff

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — task files may be stale; verify from code state.

Produce a durable handoff document that lets the next session (or a fresh agent) pick up exactly where this one stopped — without re-reading the full conversation.

## Step 1: Capture current task state

Follow `~/.claude/skills/you-got-skills/skills/shared/tracker.md` to read task state:

```bash
ls tasks/in-progress/*.md 2>/dev/null
ls tasks/backlog/*.md 2>/dev/null | head -10
```

For each in-progress task: read it fully. Note what's done, what's blocked, and what the next concrete action is.

## Step 2: Capture key decisions made this session

From the conversation context and any modified files, identify:
- Architectural or design decisions made (and why)
- Scope changes or deferrals agreed with the user
- Open questions that arose but weren't resolved
- ADRs created or updated

## Step 3: Capture the working context

```bash
# What changed this session?
git diff --stat HEAD 2>/dev/null || git status --short 2>/dev/null

# What was recently completed?
ls tasks/done/*.md 2>/dev/null | tail -5
```

## Step 4: Write the handoff doc

Write to `tasks/handoff-<YYYY-MM-DD>.md` (or user-specified path):

```markdown
# Session Handoff — <date>

## In Progress

For each open task:
- **Task:** <title>
- **Status:** What's done / what's not
- **Next action:** Exact first step for next session
- **Blocked on:** (if applicable)

## Decisions Made

- <Decision> — <Why> — <ADR reference if created>

## Open Questions

- <Question> — <Context needed to answer it>

## Suggested next session start

`/<skill>` — <why this is the right next step>
```

## Completion

Report **DONE** with the path to the handoff doc.

Suggest: Start next session by reading `tasks/handoff-<date>.md` before any skill invocation.
