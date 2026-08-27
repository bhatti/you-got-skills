---
name: ygs-interview
description: One-question-at-a-time intent extraction for underspecified asks. Closes the want-vs-stated-want gap before any plan, spec, or code is written. Use when an ask is vague, missing success criteria, or conventional rather than specific.
argument-hint: "[topic or paste the ask]"
---

# Interview

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — you own the quality of the intent you extract.

The cheapest moment to close the gap between what someone says they want and what they actually want is before any plan, spec, or code exists. Every downstream skill (ygs-refine-prd, ygs-wbs, ygs-implement) inherits the quality of the intent defined here.

## When NOT to use

- The ask is unambiguous and self-contained ("fix the null pointer in AuthService.login") — act directly
- Pure information request ("what does this function do?") — answer directly
- Mechanical operation with no design decisions ("run the tests") — proceed
- User explicitly asked for speed ("just start, we can refine") — respect that
- Async/CI context — this skill requires a live interactive user; skip it in automated runs

## Step 1: Hypothesize before asking anything

Before asking a single question, form a hypothesis about what the user actually wants:

```
Hypothesis: [what I think they want]
Confidence: [0-100%]
Gap: [what's missing that would close the gap to 95%+]
```

State this hypothesis aloud. Users often correct the framing immediately — saving multiple rounds of questioning.

If confidence is already ≥ 95%, restate their intent in the six-field format (Step 4) and ask for confirmation directly. Skip the questioning loop.

## Step 2: One question at a time

Ask one question at a time. Each question must:
- Include a **GUESS** — your best prediction of the answer: `"My guess is X — correct me if wrong."`
- Address the most important unknown first
- Be specific, not open-ended: "Who is the primary user?" not "Tell me more"

Wait for the response before asking the next question.

**Stop condition:** When you can predict their reaction to the next three questions you would ask — you are at 95%+ confidence. Stop and go to Step 4.

## Step 3: Listen for want vs. should-want signals

These patterns indicate the user is stating what they think they should want, not what they actually want:
- Buzzword goals: "we need to be more scalable / AI-native / data-driven"
- Convention-signaling answers: "best practices say we should..."
- Sophistication posturing: "we're thinking microservices / event-driven / serverless"

When you hear these: probe with — *"If you didn't have to justify this to anyone, what would you actually want?"*

The answer to that question is usually much more actionable.

## Step 4: Restate confirmed intent

Once you reach 95% confidence, restate in this six-field format:

```
**Outcome:** [what success looks like — measurable if possible]
**User:** [who this is for — specific, not "users in general"]
**Why now:** [what changed or is forcing this decision now]
**Success:** [how we'll know it worked — concrete, not "it feels better"]
**Constraint:** [hard limits — time, resources, technology, regulatory]
**Out of scope:** [what we are explicitly NOT doing — this field is non-negotiable]
```

Require **explicit confirmation** — not "sounds good" or "whatever you think." If you don't get a clear yes, ask again.

## Step 5: Output

After explicit confirmation, offer to write the confirmed intent to `docs/intent/[topic].md`.

The intent doc is the input to downstream skills:
- `/ygs-refine-prd` — to write a full PRD from this intent
- `/ygs-spike` — if feasibility of the approach is still uncertain
- `/ygs-estimate` — if timeline questions need answering before committing

Report **DONE** with the confirmed six-field intent.

---

## Common Rationalizations

| Rationalization | Reality |
|----------------|---------|
| "The ask is clear enough, I'll start" | "Clear enough" produces correct implementations of the wrong thing. Spend 5 minutes now or 5 days backtracking later. |
| "I'll ask all my questions at once to save time" | Simultaneous questions get simultaneous answers. One question gets a real answer. |
| "They said what they want, I should trust that" | People state what they think they should want. The probe in Step 3 exists for exactly this. |
| "I'll skip the Out of Scope field, it's obvious" | Nothing is obvious until it's written. The Out of Scope field prevents the most expensive surprises. |
| "My guess is probably wrong, I won't share it" | A wrong guess corrects faster than an open question. Share the guess. |
