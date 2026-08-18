---
name: ygs-review-trd
description: Review a TRD for architectural soundness, testability, operational readiness, and risk. Use after creating a TRD.
---

# Review TRD

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — you own the review quality.

## Step 1: Find the TRD

If the user specified a file, read it. Otherwise:

Follow `~/.claude/skills/you-got-skills/skills/shared/docs-discovery.md` to locate the TRD and linked PRD.

Read `~/.claude/skills/you-got-skills/templates/trd.md` and `~/.claude/skills/you-got-skills/templates/design-doc.md` as structural references for completeness checking.

## Step 2: Alignment check

- Does the design satisfy all P0 requirements from the PRD?
- Are any P0 requirements unaddressed or deferred without justification?
- Does the design introduce scope not in the PRD?

## Step 3: Architecture review

### Proportionality
- Is complexity justified by the problem? (Over-engineering without data?)
- Could a simpler approach satisfy the same requirements?
- Is abstraction warranted or premature? (Multiple consumers, or just one?)

### Design principles
For each architectural choice, classify as:
- **MUST** — Violating this blocks approval (security, data integrity, backwards compat)
- **SHOULD** — Strong recommendation with clear rationale
- **MAY** — Suggestion, take or leave

### Testability
- Can components be tested with real implementations (not mocks of internal code)?
- Are boundaries clean enough for dependency injection?
- Is stateful logic behind interfaces?
- Can the system be tested end-to-end without external paid services?

### Operational readiness
- Observability: logging, metrics, tracing for key operations?
- Failure modes: what happens when each component fails?
- Capacity: where does it break under 10x load?
- Rollback: can this be reversed without data loss?

## Step 4: Risk assessment

- Single points of failure?
- Data migration risks?
- Dependencies on external services without fallback?
- Deployment ordering constraints?

## Step 5: EARS traceability

The TRD should close the loop opened by the PRD's EARS requirements. For each P0/P1 requirement in the PRD:
- Is there a named component or module in the TRD that satisfies it?
- If the requirement is event-driven (When), does the design show what handles that event?
- If the requirement has an unwanted-behaviour counterpart (If/Then), does the design address the failure mode?

Missing traceability means the TRD may be architecturally complete but functionally incomplete — it doesn't prove the system will satisfy the requirements it was written to implement.

Reference: `~/.claude/skills/you-got-skills/skills/shared/ears-patterns.md`

## Step 6: Report

- **Strengths** — Solid design choices
- **MUST fix** — Blockers (severity: blocks approval)
- **SHOULD fix** — Strong recommendations
- **MAY improve** — Suggestions
- **Questions** — Ambiguities needing resolution
- **Recommendation** — Approve / Revise / Rethink

Report **DONE** or **DONE_WITH_CONCERNS**.
