---
name: ygs-review-prd
description: Review and critique a PRD for completeness, clarity, and feasibility. Use after creating or receiving a PRD.
---

# Review PRD

## Step 1: Find the PRD

If the user specified a file, read it. Otherwise:

```bash
ls -t docs/prd/*.md 2>/dev/null | head -5
```

Ask the user which PRD to review if multiple exist.

## Step 2: Review against criteria

Evaluate the PRD on:

1. **Problem clarity** — Is the problem statement specific and compelling?
2. **User definition** — Is the target user well-defined (not "everyone")?
3. **Success criteria** — Are they measurable and time-bound?
4. **Requirements** — Are P0/P1/P2 clearly prioritized? Any P0 that should be P1?
5. **Scope** — Are non-goals explicit? Is scope creep visible?
6. **Feasibility** — Are there technical or resource blockers?
7. **Dependencies** — Are external dependencies identified?
8. **Gaps** — Missing user stories? Unasked questions?

## Step 3: Report

Produce a structured review with:
- **Strengths** — What's well done
- **Issues** — Problems to fix (severity: blocker/major/minor)
- **Questions** — Things that need answers before proceeding
- **Recommendation** — Approve / Revise / Rethink

Report **DONE** or **DONE_WITH_CONCERNS** based on findings.
