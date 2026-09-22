# Implementation Confidence Rubric

Shared by: `ygs-enrich-ticket`

Used to score how confident the agent is that a generated implementation plan is accurate and useful before writing it to a ticket. Score 0–100.

## Scoring

### Factor 1: Repo evidence (0–45 points)

How well does the codebase confirm the approach?

| Evidence found | Points |
|----------------|--------|
| Keyword match (type name, function name, module path) found in codebase | +15 |
| Direct function/type hit — found the exact entry point the ticket refers to | +30 |
| End-to-end trace — followed the call chain from entry point to data layer | +45 |

Take the **highest** single value that applies (not cumulative).

### Factor 2: Ticket fit (0–25 points)

How complete is the ticket as a spec?

| Ticket state | Points |
|-------------|--------|
| Vague description, no clear scope | +0 |
| Clear scope with named components or modules | +15 |
| Spec + acceptance criteria both present | +25 |

### Factor 3: Blast radius penalty (0 to −30 points)

How wide is the change?

| Blast radius | Penalty |
|-------------|---------|
| Single file or single module | −0 |
| Cross-module (2+ packages/directories) | −15 |
| Cross-service, schema change, or public API change | −30 |

## Final score = Factor 1 + Factor 2 + Factor 3

## Decision thresholds

| Score | Confidence | Action |
|-------|------------|--------|
| ≥ 70 | HIGH | Write plan to ticket description |
| 40–69 | MEDIUM | Write plan with caveat header: "Confidence: MEDIUM — verify file paths and approach before implementing." |
| < 40 | LOW | Do NOT write. Flag the ticket for human review with a comment explaining what's missing. |

## Caveat header format (MEDIUM)

```
> **AI-generated implementation plan — Confidence: MEDIUM**
> The plan was generated with incomplete codebase evidence. Verify that the named
> files and functions exist before handing off to an agent.
```
