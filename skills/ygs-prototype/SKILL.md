---
name: ygs-prototype
description: >
  Use when the right design can't be settled in the abstract — build something
  concrete to react to. Two modes: logic/state model (interactive HTML state machine)
  or UI look (multiple visually distinct variants). Not for proving technical
  feasibility — use ygs-spike for that.
argument-hint: "[<design-question>] [--mode logic|ui]"
---

# Prototype

A prototype answers a specific design question by building something concrete to react to. It is a communication tool, not a technical proof. Use it when arguing in the abstract has stopped producing alignment.

**The question comes first. The prototype is evidence for an answer — not a solution.**

## When NOT to use

- The question is about technical feasibility ("can we do X?") — use `/ygs-spike`
- The design is already agreed on — just implement it
- No specific question exists — define the question before building; a prototype without a question is just throwaway code

## Difference from a spike

| | Prototype | Spike |
|-|-----------|-------|
| Question | "Should it look/behave like this?" | "Can we do this?" |
| Audience | Stakeholders, designers, team | Engineering |
| Output | Artifact to react to | Benchmark / feasibility proof |
| Code quality | Throwaway | Throwaway |
| Where it lives | Branch, never merges | Branch, never merges |

## Step 1: State the design question

The question goes at the top of the prototype as the title or header — not a description of what it does, but the question it answers. Examples:
- "Should the onboarding flow use a wizard or a single long form?"
- "Does a state machine model feel right for the checkout flow, or is it overconstrained?"
- "Which of these three dashboard layouts surfaces the most important data above the fold?"

Elicit the question if not provided. Do not proceed without a specific question.

## Step 2: Select mode

`--mode logic` — for questions about state models, data flow, FSM design, or domain object behavior. Default when the question is about how the system behaves.

`--mode ui` — for questions about layout, visual hierarchy, interaction patterns, or which of several approaches to present. Default when the question is about what users see.

If mode is ambiguous, ask: "Is this question about how the system behaves, or about how it looks and feels?"

## Step 3a: Logic mode — interactive state machine

Build a single self-contained HTML file:
- No build step, no external CDN dependencies — runs with `open <file>` or `python3 -m http.server`
- The design question as the page `<title>` and a visible `<h1>`
- Render the state machine: nodes as boxes, transitions as arrows or buttons
- Every valid state transition is clickable — clicking advances the machine
- Current state displayed prominently
- Invalid transitions shown as disabled (greyed out), not hidden

The goal is for a reader to click through the states and say "yes, this is how I'd expect it to work" or "no, this transition doesn't make sense."

## Step 3b: UI mode — variant comparison

Build 2–3 variants that are radically different from each other — not colour tweaks, but structurally different layouts or interaction patterns:

Option A: Separate HTML files (`variant-a.html`, `variant-b.html`, `variant-c.html`)
Option B: Single file with `?variant=1`, `?variant=2`, `?variant=3` query param — preferred for quick switching

Each variant must:
- Be functionally equivalent (same content, different presentation)
- Be fully static (no server needed)
- Have the design question visible at the top

The goal is to force a concrete choice. If variants look similar, they're not different enough — make them more extreme.

## Step 4: Place and mark as throwaway

Place the prototype near the relevant production code (not in `dist/`, `tmp/`, or the project root):

```
src/checkout/
  checkout.ts         ← production code
  prototype-state-model.html   ← prototype lives here
```

Header comment in the file:

```html
<!-- THROWAWAY PROTOTYPE
     Question: [state the question]
     Status: [OPEN / RESOLVED: <answer>]
     Do not merge to main. Remove after decision is made.
-->
```

## Step 5: Run and verify

```bash
open <file>              # macOS
xdg-open <file>          # Linux
python3 -m http.server   # if file uses relative paths
```

Verify: every interactive element responds, the question is visible, the states/variants are meaningfully different.

## Step 6: Capture the decision

After the prototype is reviewed (now or in a separate session), capture the decision:

```
Decision: [the question]
Answer:   [what was decided]
Evidence: [which variant won / which state model was correct / what was wrong about the others]
```

Write this as a comment on the originating issue, or as a new ADR in `docs/adr/`. Update the prototype header `Status` to `RESOLVED: <answer>`.

## Completion

Report **DONE** with:
- File(s) created
- Design question
- How to open/run the prototype
- Next step: share with stakeholders or schedule review

Suggest: `/ygs-refine-architecture` if the decision is architectural, `/ygs-spike` if technical feasibility is still open after the design question is answered.

**Note:** Prototype stays on the current branch. Do not merge to main. Once the decision is captured, the prototype can be deleted.
