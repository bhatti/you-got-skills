---
name: ygs-wayfinder
description: >
  Use when an effort is too large and fog-bound for a PRD or spike — the destination
  is known but the path through it isn't. Creates a decision map on the tracker and
  resolves one decision per session until the path is clear. Not for scoped tasks;
  use ygs-spike or ygs-refine-prd instead.
argument-hint: "[<destination-statement>] [--map-ticket <url>]"
---

# Wayfinder

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.

**Iron law: one session = one ticket resolved. Never implement during a wayfinder session.**

A wayfinder session is navigation, not construction. You are clearing fog, not building the road. The moment you start implementing, you've stopped navigating — and you're now committed to a path you haven't finished charting.

## When NOT to use

- The work is already scoped to a PRD or TRD — use `/ygs-refine-prd` or `/ygs-wbs`
- There's a specific unknown to validate — use `/ygs-spike`
- The destination itself is unclear — use `/ygs-interview` or `/ygs-brainstorm` first; wayfinder requires a known destination

## Step 1: State the destination

The destination is a one-sentence statement of the end state: "Users can do X without Y friction."

If no destination was provided, elicit it:
- "What does success look like when this effort is done?"
- "What can someone do that they can't do today?"

Do not proceed until the destination is stated in one sentence. A fuzzy destination produces a fuzzy map.

## Step 2: Load or create the map ticket

The map ticket is the single source of truth for this effort. Label it `wayfinder:map`.

**If `--map-ticket <url>` is provided:** Fetch and read the existing map ticket. Review Decisions Made, Fog of War, and Out of Scope.

**Otherwise, create a new map ticket** with this structure:

```markdown
## Destination
[One sentence]

## Decisions Made
[empty — filled in as sessions complete]

## Fog of War
[Things you know are coming but can't phrase as actionable tickets yet]

## Out of Scope
[Explicit exclusions — things that would be pulled in by gravity but shouldn't be]
```

Using `~/.claude/skills/you-got-skills/skills/shared/tracker.md` to create/fetch the ticket.

## Step 3: Populate the fog

List the known unknowns — things you're aware of but can't resolve yet. These go in the Fog of War section. Examples:
- "We don't know if the existing auth flow supports multi-tenant scoping"
- "Unknown whether the current DB schema can support the new query pattern"
- "Not sure how the mobile client handles the new event type"

This step is generative — the goal is to surface as much fog as possible, not to resolve it.

## Step 4: Create decision tickets

Turn fog items into decision tickets. Each ticket must be:
- ONE resolvable question (not a vague exploration)
- Typed with one of these labels:
  - `wayfinder:grilling` — answered via a structured Q&A session with a stakeholder (HITL)
  - `wayfinder:prototype` — answered by building a throwaway artifact (HITL, see `/ygs-prototype`)
  - `wayfinder:research` — answered by reading code, docs, or running a test (AFK)
  - `wayfinder:task` — a small concrete action that resolves the uncertainty (either)

Ticket title format: "Decision: [the question]"

Using `~/.claude/skills/you-got-skills/skills/shared/tracker.md` to create tickets.

## Step 5: Identify the frontier

**Frontier** = tickets that are: open + unblocked + unclaimed.

Sort by blocking order — tickets that block other decisions come first.

Present the frontier to the user. Confirm which one to work on this session.

## Step 6: Resolve one frontier ticket

Claim the ticket (assign to self). Work it according to its type:

- **grilling**: Run a structured questioning session. Capture the answer in a resolution comment.
- **prototype**: Invoke `/ygs-prototype`. The prototype outcome IS the resolution.
- **research**: Read code, docs, or run a test. Produce a finding. Do not generalize beyond the question.
- **task**: Execute the task. The outcome IS the resolution.

Post the resolution as a comment on the ticket. Close the ticket. Update the map ticket:
- Move the resolved question from Fog of War to Decisions Made (with the answer)

## Step 7: Re-evaluate and hand off

Re-read the map ticket. Is the path from here to the destination clear enough to write a PRD or decompose into tasks?

**If yes:** Declare the wayfinder complete. Hand off to `/ygs-refine-prd` or `/ygs-wbs`. Note the Decisions Made section as the context to pass forward.

**If no:** Return to Step 5 for the next session.

## Completion

Report **DONE** with:
- Map ticket URL
- Decision ticket resolved this session (question + answer)
- Current fog count (remaining open decisions)
- Whether the path is clear enough for handoff

Suggest: `/ygs-refine-prd` or `/ygs-wbs` when map clears.

---

## Common Rationalizations

| Rationalization | Reality |
|----------------|---------|
| "We can figure it out as we go" | Fog-of-war items discovered mid-sprint become sprint blockers. The cost is not the research — it's the blocked team members waiting for answers. |
| "We already know enough to start" | The items in your fog cost the same to resolve before sprint planning as after. They cost much more after if they block a half-finished implementation. |
| "This is too heavyweight for a small effort" | If the path is clear, this skill does nothing — you'll skip Step 3 (empty fog). The ceremony scales with the fog, not the effort size. |
| "Let's just spike it" | A spike proves technical feasibility. Wayfinder resolves requirements ambiguity and stakeholder alignment. Different questions. |
