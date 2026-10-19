# Implementation Confidence Rubric

Shared by: `ygs-enrich-ticket`. Statuses and write rules: `ticket-readiness.md`.

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
| Spec + acceptance criteria both present (score before writing ACs; ACs you generated yourself do not count) | +25 |

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
| ≥ 70 | HIGH | Write the plan as the generated section (see `ticket-readiness.md`) |
| 40–69 | MEDIUM | Write the plan with the caveat blockquote below as its first lines |
| < 40 | LOW | Do NOT write to the description. Post one comment saying what is missing. The ticket becomes `IMPL_PLAN_LOW_CONFIDENCE` so it is not enriched again until a person acts. |

## Caveat header format (MEDIUM)

```
> **AI-generated implementation plan — Confidence: MEDIUM**
> The plan was generated with incomplete codebase evidence. Verify that the named
> files and functions exist before handing off to an agent.
```
