---
name: ygs-brainstorm
description: Design gate before writing code — classify the work (spike/bounded/architectural), explore the approach, and get approval before any implementation. Use when you know what to build but not how. Triggers before new features, new components, non-trivial modifications to existing behavior.
argument-hint: "[topic or paste the requirement]"
---

# Brainstorm

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — you own the design decision, not just the code.

**If intent is still unclear** (you don't know what to build), run `/ygs-interview` first. This skill assumes intent is confirmed and asks: how should we build it?

## Step 1: Classify the work

Classify before exploring. The path depends on the type of change.

**Spike** — Feasibility is still uncertain. You don't know if the approach will work, what the API looks like, or whether performance is achievable.
→ Proceed to `/ygs-spike`. Return here after to choose an implementation path.

**Bounded** — Scoped change to existing code. Fewer than 3 files affected. Clear interface. No new abstractions. No schema or API changes.
→ Proceed to Step 2 (fast path).

**Architectural** — New subsystem, new abstraction, cross-cutting concern, multiple components, schema change, new public API, or a change that will constrain future decisions.
→ Proceed to Step 2 (deep path).

State the classification before proceeding.

## Step 2 (Bounded fast path)

State your proposed approach in 2-3 sentences:
- What changes and what doesn't
- Why this approach over the most obvious alternative
- What the failure mode is if the approach is wrong

Require **explicit user confirmation** before proceeding. "Sounds fine" counts. Silence does not.

Then: "Run `/ygs-implement`."

## Step 2 (Architectural deep path)

Explore the design with 3-5 questions — **one at a time**, with a GUESS each time:

1. **Module boundary** — Where does this capability live? Is it a new module or an extension of an existing one? *My guess is X — correct me if wrong.*
2. **Interface** — What does the caller see vs. what's hidden? What does the minimal public surface look like?
3. **Data model** — What are the core entities and their relationships? What state must be stored vs. derivable?
4. **Failure modes** — What breaks under load? Partial failure? Concurrent access? What's the worst-case data loss scenario?
5. **Alternatives** — What's the next-best design? Why is the proposed approach better?

Stop asking when you can articulate the design direction in a paragraph without uncertainty.

**After questions:**

Summarize the design direction in 3-5 sentences. Write it to `docs/design/<topic>.md`.

Then recommend the appropriate depth:
- For API surface / cross-service interface changes → `/ygs-refine-trd`
- For system-wide changes / new components → `/ygs-refine-architecture`
- For well-bounded architectural changes with clear constraints → proceed to `/ygs-implement` directly

**Require explicit user approval before recommending any downstream action.**

## Hard Gate

No implementation starts without explicit user approval of the design direction.

If you are tempted to skip this and "just start coding" — that temptation is the signal that this skill is most needed.

## Common Rationalizations

| Rationalization | Reality |
|----------------|---------|
| "It's a small change, I'll just code it" | Bounded changes need 2 sentences of design. Architectural changes need 20 minutes of questions. The cost of skipping is proportional to how "small" you thought it was. |
| "I'll figure it out as I go" | Design questions cost nothing. Undoing code costs hours. Ask the question now. |
| "The requirements are clear enough" | Requirements say what. This skill asks how. The "how" contains half the bugs. |
| "We've done similar things before" | Similar is not the same. The difference between similar and same is where the bug lives. |
| "The user wants to move fast" | A 10-minute design conversation prevents a 4-hour backtrack. Moving fast on the wrong design is not fast. |

---

References:
- `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`
- `~/.claude/skills/you-got-skills/skills/shared/functional-design.md`
- `~/.claude/skills/you-got-skills/templates/design-doc.md`
