---
name: ygs-uat
description: User Acceptance Testing from the customer perspective. Validates user stories and end-to-end workflows.
---

# User Acceptance Testing

## Step 1: Gather user stories

Read the source PRD for user stories and success criteria:

```bash
ls -t docs/prd/*.md 2>/dev/null | head -3
```

If no PRD, ask the user: What should the end user be able to do?

## Step 2: Define acceptance scenarios

For each user story, define:
- **Given** — Starting state / preconditions
- **When** — User action
- **Then** — Expected outcome

Focus on the customer's perspective, not internal implementation.

## Step 3: Validate workflows

For each scenario:
- Trace the end-to-end user flow through the code
- Check that the user gets appropriate feedback at each step
- Verify error messages are user-friendly (not technical jargon)
- Confirm the user can recover from mistakes

## Step 4: Assess user experience

- Is the flow intuitive? Minimum steps to complete?
- Are there dead ends or confusing states?
- Does it handle real-world data (long names, special characters, etc.)?
- Is performance acceptable from user's perspective?

## Step 5: Report

For each user story:
- **PASS** — Meets acceptance criteria
- **FAIL** — Doesn't meet criteria (describe gap)
- **PARTIAL** — Works but with friction

Overall assessment:
- Ready for release?
- Blockers for user satisfaction?
- Suggested improvements (prioritized)

Report **DONE** or **DONE_WITH_CONCERNS**.
