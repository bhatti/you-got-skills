---
name: ygs-review-prd
description: Review and critique a PRD for completeness, clarity, and feasibility. Use after creating or receiving a PRD.
---

# Review PRD

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — you own the review quality.

## Step 1: Find the PRD

If the user specified a file, read it. Otherwise:

Follow `~/.claude/skills/you-got-skills/skills/shared/docs-discovery.md` to locate the PRD.

Ask the user which PRD to review if multiple exist.

## Step 2: Review against criteria

Evaluate the PRD on:

1. **Problem clarity** — Is the problem statement specific and compelling?
2. **User definition** — Is the target user well-defined (not "everyone")?
3. **Success criteria** — Are they measurable and time-bound?
4. **Requirements** — Are P0/P1/P2 clearly prioritized? Any P0 that should be P1?
5. **EARS compliance** — Do behavioral requirements use a named EARS pattern? (see `shared/ears-patterns.md`)
   - Are `WHEN` (event-driven) and `WHILE` (state-driven) requirements present where expected?
   - Are `IF/THEN` (unwanted behaviour) requirements present for every MUST? Missing failure modes = incomplete.
   - Are optional features marked `WHERE`, not buried as inline conditionals?
   - Does any requirement match two patterns? If so, flag it to split.
6. **Scope** — Are non-goals explicit? Is scope creep visible?
7. **Feasibility** — Are there technical or resource blockers?
8. **Dependencies** — Are external dependencies identified?
9. **Gaps** — Missing user stories? Unasked questions?

## Step 3: Report

Produce a structured review with:
- **Strengths** — What's well done
- **Issues** — Problems to fix (severity: blocker/major/minor)
- **Questions** — Things that need answers before proceeding
- **Recommendation** — Approve / Revise / Rethink

Report **DONE** or **DONE_WITH_CONCERNS** based on findings.

**Suggest next steps:**
- If MUST items found → suggest `/ygs-refine-prd` to resolve them before approval
- If approved → suggest `/ygs-wbs` to decompose into tasks, or `/ygs-estimate` for timeline
