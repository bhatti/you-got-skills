---
name: ygs-refine-prd
description: Refine a Product Requirements Document through structured questioning — challenges assumptions, sharpens language, resolves ambiguity until shared understanding. Best with reasoning model (Opus-class).
---

# Refine PRD

For shared refinement protocol (questioning, challenges, CONTEXT.md, ADRs), read `~/.claude/skills/you-got-skills/skills/shared/refine-scaffold.md`.

## Approach

Interview the user relentlessly about every aspect of their product requirements until reaching shared understanding. Walk down each branch of the decision tree, resolving dependencies one-by-one.

Follow the questioning protocol from `shared/refine-scaffold.md`.

## Step 1: Find or receive the PRD

If the user specified a file, read it. Otherwise:

```bash
ls -t docs/prd/*.md 2>/dev/null | head -5
```

If no PRD exists, work from whatever the user provides (rough notes, verbal description, conversation context).

## Step 2: Read template for structure

Read `~/.claude/skills/you-got-skills/templates/prd.md` as structural reference.

## Step 3: Refine through questioning

Walk through each dimension, challenging vagueness and resolving ambiguity:

### Problem clarity
- What specifically is broken or missing today?
- Who told you this is a problem? How do you know?
- What happens if you do nothing?

### User definition
- Who is the target user? (Not "everyone" — be specific)
- What's their current workflow without this feature?
- How many users are affected?

### Success criteria
- How will you measure success? (Specific metrics, not "users are happy")
- What's the timeline for measuring?
- What would make you declare this a failure?

### Requirements
- For each stated requirement: Why is this P0 and not P1? What breaks without it?
- Is there a simpler version that delivers 80% of the value?
- What did you explicitly decide NOT to include? Why?

### Capabilities mapping
- What's genuinely new vs. what's modifying existing behavior?
- Are any existing capabilities being removed? Is that a breaking change?
- Is the scope of "modified" larger than "new"? (brownfield signal — important for estimation)

### Scenarios and acceptance
- For each requirement, define at least one Given/When/Then scenario
- Push for edge case scenarios: what happens with empty input? At scale? On failure?
- Use RFC 2119 keywords for requirement strength: MUST (P0), SHOULD (P1), MAY (P2)
- For behavioral requirements, use EARS notation (see `shared/ears-patterns.md`):
  - `THE SYSTEM SHALL [behavior]`                                    (ubiquitous — always active)
  - `WHEN [trigger], THE SYSTEM SHALL [response]`                    (event-driven)
  - `WHILE [state], THE SYSTEM SHALL [behavior]`                     (state-driven)
  - `WHERE [feature is included], THE SYSTEM SHALL [behavior]`       (optional feature)
  - `IF [fault/unwanted trigger], THEN THE SYSTEM SHALL [response]`  (unwanted behaviour)
  - `WHILE [state], WHEN [trigger], THE SYSTEM SHALL [response]`     (complex — state + event)
- Validate coverage: every MUST requirement has at least one IF/THEN covering its failure mode
- Check for missing patterns: optional features (`WHERE`) and unwanted behaviours (`IF/THEN`) are the most commonly skipped

### Constraints and impact
- What's the worst that can happen if this goes wrong?
- What assumptions are you making about user behavior?
- Are there compliance, performance, or compatibility constraints?
- What systems, APIs, or teams are affected?

Follow the challenge patterns from `shared/refine-scaffold.md` (vague language, terminology conflicts, code contradictions, concrete scenarios).

Additional PRD-specific constraint: Do NOT conflate SHOULD (P1) with MUST (P0) — ask which it is.

## Step 4: Write the refined PRD

Use directory convention detection from `shared/refine-scaffold.md` (type = `prd`).

Write (or update) the PRD. Fill all sections. Mark unresolved items in Open Questions.

## Step 5: Completion

Report **DONE** with the file path.
Suggest: `/ygs-review-prd` for independent critique, `/ygs-refine-trd` for technical design.
