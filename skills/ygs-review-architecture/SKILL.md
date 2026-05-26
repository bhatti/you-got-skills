---
name: ygs-review-architecture
description: Review architecture documents for module depth, testability, operational readiness, and proportionality. Uses deep-module vocabulary. Supports parallel independent reviewers with consensus synthesis. Use for significant system design changes.
argument-hint: "[file-path] [--parallel N]"
---

# Review Architecture

## Vocabulary

- **Deep module** = small interface + lots of implementation (high leverage — good)
- **Shallow module** = large interface + thin implementation (pass-through — bad)
- **Seam** = where an interface lives; behavior changes without editing in place
- **Deletion test** = imagine deleting the module. Complexity vanishes? Pass-through. Reappears in N callers? Earning its keep.

## Arguments

Parse arguments for:
- File path (required — architecture doc to review)
- `--parallel N` — run N independent reviewers and synthesize by consensus (default: 1)

## Step 1: Find the architecture doc

If not specified:

```bash
ls docs/architecture/*.md 2>/dev/null
ls docs/design/*.md 2>/dev/null
```

Read the document and any linked PRDs/TRDs.

Read `~/.claude/skills/you-got-skills/templates/design-doc.md` as structural reference for completeness checking.

## Step 2: Module depth analysis

For each proposed module/component:
- Apply the **deletion test**: would removing it concentrate complexity or disperse it?
- Is it **deep** (small interface, rich implementation) or **shallow** (many methods, little logic)?
- Are seams real (2+ adapters) or hypothetical (1 adapter)?
- Could shallow modules be collapsed into their callers?

## Step 3: Proportionality analysis

- Is the complexity justified by quantitative data (load numbers, user counts, SLA requirements)?
- Could a simpler architecture satisfy the stated requirements?
- Are abstractions warranted by multiple consumers, or premature?
- What's the operational cost of this architecture vs simpler alternatives?

## Step 4: Architecture principles (MUST/SHOULD/MAY)

### MUST (blocks approval if violated)
- Security: trust boundaries properly defined, auth model sound
- Data integrity: no paths that corrupt or lose data
- Backwards compatibility: existing clients/consumers not broken
- Failure isolation: cascading failure paths eliminated

### SHOULD (strong recommendation)
- Testability: components testable with real implementations, clean DI boundaries
- Observability: key operations instrumented, failures diagnosable from logs alone
- Scalability: identified bottlenecks with mitigation path
- Consistency: aligns with existing system patterns unless justified deviation

### MAY (suggestion)
- Performance optimizations for non-critical paths
- Alternative technology choices within the same category
- Naming and organizational preferences

## Step 5: Operational review

1. **Failure modes** — For each component: what breaks, blast radius, recovery path
2. **Stateful components** — Migration path, backwards-compatible schema, data durability
3. **Dependencies** — External services have timeout/retry/circuit-breaker?
4. **Capacity planning** — Where does it break at 10x? At 100x?
5. **Deployment** — Can old and new coexist? Rolling deploy safe? Rollback path?
6. **On-call impact** — New alerts, runbooks, dashboards needed?

## Step 6: Testability review

- Can the system be integration-tested without paid external services?
- Are boundaries clean enough for stubs at external edges only?
- Can failure scenarios be reproduced in test?

## Step 7: Report

### Single-pass mode (N=1, default)

For each finding:
- **Severity:** MUST | SHOULD | MAY
- **Category:** Proportionality | Security | Operations | Testability | Consistency
- **Description** and recommended change

### Parallel mode (N>1)

Run N independent review passes (Steps 2-5) then synthesize:

1. Match findings across reviewers by Category + topic similarity
2. Weight by agreement:
   - N/N agree → **HIGH** confidence (present first)
   - Majority agree → **MEDIUM** confidence
   - 1 only → **LOW** confidence (note as single-reviewer finding)
   - Contradictions → **SPLIT** (flag for human judgment)
3. Deduplicate: show each finding once with its confidence level
4. Merge verdict: use the most conservative recommendation across reviewers

### Proportionality rating

State one of:
- **Proportionate** — complexity justified by stated requirements
- **Over-engineered** — simpler architecture satisfies requirements (explain what to cut)
- **Justified complexity** — looks heavy but specific constraints demand it (cite which)

### Output format

| # | Severity | Category | Confidence | Finding | Recommendation |
|---|----------|----------|------------|---------|----------------|

**Escalation triggers** (require explicit human decision):
- Deliberate deviation from a MUST-level principle
- Complexity without quantitative justification
- Breaking change to existing consumers without migration path

After presenting findings, ask:
- For each MUST: fix now, defer with justification, or disagree?
- For each SHOULD: fix now, defer, or skip?
- SUGGESTIONs: "Accept all? Or review individually?"

Overall: **Approve** / **Revise** / **Rethink**
Report **DONE** or **DONE_WITH_CONCERNS**.
