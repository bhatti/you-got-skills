---
name: ygs-retro
description: Retrospective on recent work — what went well, what didn't, what to change. Use after completing a feature or sprint.
---

# Retrospective

## Step 1: Gather context

Review recent work:

```bash
git log --oneline --since="1 week ago" 2>/dev/null || git log --oneline -20
```

Check completed tasks:

```bash
ls tasks/done/*.md 2>/dev/null
```

Read recent PRDs/TRDs for what was planned vs delivered.

## Step 2: Analyze

Assess across these dimensions:

### What went well
- Features delivered on time?
- Code quality (test coverage, clean reviews)?
- Good architectural decisions?
- Effective collaboration?

### What didn't go well
- Missed estimates? Why?
- Bugs shipped? Root causes?
- Scope creep? Where did it come from?
- Blocked time? What caused delays?

### Patterns
- Recurring issues across tasks?
- Areas of the codebase that consistently cause problems?
- Process friction (reviews too slow, unclear requirements)?

## Step 3: Recommendations

Produce 2-3 specific, actionable changes:
- **Keep doing:** Practices that worked well
- **Start doing:** New practices to try
- **Stop doing:** Practices causing friction

Each recommendation should be concrete (not "communicate better" — instead "add acceptance criteria to every task before implementation starts").

## Step 4: Incidents

If any incidents occurred during the period, use `references/postmortem-template.md` for structured post-mortem analysis with Five Whys and action items.

## Step 5: Completion

Report **DONE** with the retrospective summary.
Suggest saving as `docs/retro/YYYY-MM-DD.md` if the team wants to track retros.
