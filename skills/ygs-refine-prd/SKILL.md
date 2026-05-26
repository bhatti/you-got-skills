---
name: ygs-refine-prd
description: Refine a Product Requirements Document through structured questioning — challenges assumptions, sharpens language, resolves ambiguity until shared understanding. Best with reasoning model (Opus-class).
---

# Refine PRD

## Approach

Interview the user relentlessly about every aspect of their product requirements until reaching shared understanding. Walk down each branch of the decision tree, resolving dependencies one-by-one.

**Ask questions one at a time. Provide your recommended answer for each. Wait for feedback before continuing.**

If a question can be answered by exploring the codebase, explore the codebase instead of asking.

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
- For behavioral requirements, use EARS notation for precision:
  - "WHEN [trigger], THE SYSTEM SHALL [response]" (event-driven)
  - "WHILE [state], THE SYSTEM SHALL [behavior]" (state-driven)
  - "WHERE [condition], THE SYSTEM SHALL [behavior]" (conditional)
  - "THE SYSTEM SHALL [behavior]" (ubiquitous — always true)

### Constraints and impact
- What's the worst that can happen if this goes wrong?
- What assumptions are you making about user behavior?
- Are there compliance, performance, or compatibility constraints?
- What systems, APIs, or teams are affected?

**Challenge vague language:** When the user says something ambiguous ("fast", "easy", "scalable"), push for specifics. "Fast means what? 100ms? 1 second? Faster than today?"

**Discuss concrete scenarios:** For each requirement, invent a specific scenario that probes edge cases and forces precision.

**Cross-reference with code:** When the user states how something currently works, check whether the code agrees. Surface contradictions: "You said users can cancel partial orders, but the code cancels the entire Order — which is right?"

**Challenge terminology:** When the user uses a term that conflicts with existing glossary (CONTEXT.md) or code naming, call it out. Propose canonical terms for vague/overloaded language.

### Inline documentation updates

**CONTEXT.md (domain glossary):** When a domain term is resolved during questioning, update or create `CONTEXT.md` immediately. Only domain-specific terms — not general programming concepts. One sentence definitions, list aliases to avoid.

Create lazily: if no `CONTEXT.md` exists, create one when the first term resolves.

### Do NOT (negative constraints)
- Do NOT add requirements the user didn't ask for
- Do NOT assume features from similar products should be included
- Do NOT expand scope beyond what the user describes
- Do NOT conflate SHOULD (P1) with MUST (P0) — ask which it is
- Do NOT write the PRD until questioning reaches shared understanding

## Step 4: Write the refined PRD

Determine directory convention (auto-detect from existing structure or ask):

```bash
# Check if product/project hierarchy exists
ls docs/prd/*/*/ 2>/dev/null && echo "hierarchy" || echo "flat"
mkdir -p docs/prd
```

- **Flat:** `docs/prd/YYYY-MM-DD-<slug>.md`
- **Hierarchy:** `docs/prd/{product}/{project}/YYYY-MM-DD-<slug>.md`

Write (or update) the PRD. Fill all sections. Mark unresolved items in Open Questions.

## Step 5: Completion

Report **DONE** with the file path.
Suggest: `/ygs-review-prd` for independent critique, `/ygs-refine-trd` for technical design.
