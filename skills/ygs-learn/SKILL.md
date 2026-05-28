---
name: ygs-learn
description: Capture and surface operational learnings across sessions — failures, surprises, patterns that should inform future work. Use after incidents, surprising bugs, or when a pattern keeps recurring.
---

# Learn

Persistent knowledge capture that feeds back into future sessions. Unlike `/ygs-retro` (which reviews a period of work), `/ygs-learn` captures a single atomic learning as it happens.

## Step 1: Capture the learning

Ask (or infer from context):
- **What happened?** — The specific event, bug, or surprise
- **Why does it matter?** — What would go wrong next time without this knowledge
- **Category:** Edge Case | Integration Gotcha | Performance Cliff | Security Trap | Process Friction | Domain Rule | Tooling Quirk

## Step 2: Check for duplicates

```bash
find docs/learnings/ -name "*.md" 2>/dev/null | xargs grep -l "<relevant keyword>" 2>/dev/null
```

If a similar learning exists: reinforce it (add date + additional evidence) rather than duplicating.

## Step 3: Write the learning

```bash
mkdir -p docs/learnings
```

Write to `docs/learnings/YYYY-MM-DD-<slug>.md`:

```markdown
# <Title>

**Category:** <category>
**Date:** YYYY-MM-DD
**Source:** <what triggered this — incident, review finding, spike result, etc.>

## Learning

<1-3 sentences: the rule or pattern>

## Evidence

<What happened that taught us this>

## Application

<When to apply this in future work — specific trigger conditions>
```

## Step 4: Surface relevant learnings (when invoked without a new learning)

If the user asks "what do we know about X?" or invokes without new context:

```bash
grep -rl "<keyword>" docs/learnings/ 2>/dev/null
```

Present matching learnings sorted by relevance. Note any that may be stale (older than 6 months — verify still applies).

## Step 5: Completion

Report **DONE** with:
- Learning captured (or surfaced)
- File path
- Related learnings (if any exist in the same category)

Suggest: `/ygs-retro` for broader reflection, or `/ygs-investigate` if the learning came from a bug.
