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

## Step 4a: Extract mode (after PR merge — invoked with `--extract <pr-url>`)

When a PR has been merged and review feedback is available, extract learnings from that feedback automatically:

1. Fetch PR comments via `gh pr view <pr-url> --comments` (or Bitbucket REST equivalent)
2. For each comment thread: assess whether it reveals a recurring pattern, a gotcha, or a surprise that would apply to future work
3. Skip comments that are purely stylistic, already captured, or specific to the PR with no generalizability
4. For each actionable learning: run Steps 2-3 above (deduplicate, then write)
5. Report: N learnings extracted, M skipped (already existed or not generalizable)

## Step 4b: Audit mode (invoked with `--audit`)

Scan learnings for staleness and quality:

```bash
find docs/learnings/ -name "*.md" -mtime +90 2>/dev/null
```

For each learning older than 90 days:
- Check whether the evidence still applies (the code/pattern it references may have changed)
- If stale: update with new evidence, or mark as superseded and delete
- Report: N reviewed, M updated, K deleted

## Step 5: Completion

Report **DONE** with:
- Learning captured (or surfaced)
- File path
- Related learnings (if any exist in the same category)

Suggest: `/ygs-retro` for broader reflection, or `/ygs-investigate` if the learning came from a bug.
