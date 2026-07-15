---
name: ygs-estimate
description: Complexity-based estimation — t-shirt sizing at feature level, story points at task level, capacity planning with KTLO/oncall/vacation buffers. Use before WBS or when management needs timelines.
---

# Estimate

## Step 1: Gather inputs

Read the PRD and TRD (if available):

```bash
ls -t docs/prd/*.md 2>/dev/null | head -5
ls -t docs/trd/*.md 2>/dev/null | head -5
```

If no docs exist, work from whatever the user provides.

## Step 2: Verify assumptions against code

Before estimating, check the codebase to calibrate:
- Do the modules/components the PRD/TRD references actually exist and work as described?
- What is the actual complexity of the code being modified? (Read it, don't guess from descriptions)
- Are there hidden dependencies, technical debt, or coupling that the design doesn't mention?
- Is there existing code that makes parts of this work trivial — or existing complexity that makes it harder than it sounds?

Estimates based solely on document descriptions are fiction. Ground them in code reality.

## Step 3: Feature-level t-shirt sizing

For each major capability in the PRD, assign a t-shirt size based on complexity, uncertainty, and integration surface:

| Size | Complexity | Typical Duration (1 dev) | Signal |
|------|-----------|-------------------------|--------|
| **XS** | Trivial change, well-understood | < 1 day | Config change, copy update |
| **S** | Single component, low uncertainty | 1-3 days | Bug fix, small feature in known area |
| **M** | Multiple components, moderate uncertainty | 1-2 weeks | New feature touching 3-5 modules |
| **L** | Cross-cutting, high uncertainty | 2-4 weeks | New subsystem, integration with external service |
| **XL** | Architectural change, very high uncertainty | 4-8 weeks | New service, major refactor, platform change |

**Uncertainty multipliers:**
- New technology/domain the team hasn't used → 1.5x
- External dependency with unclear API/timeline → 1.5x
- Both → 2x

## Step 4: Decompose into estimable units

Break each feature into components small enough to estimate (few days max). Use story points with Fibonacci sequence: 1, 2, 3, 5, 8, 13, 21.

| Points | Meaning |
|--------|---------|
| 1 | Trivial, no unknowns, done in hours |
| 2 | Small, well-understood, < 1 day |
| 3 | Moderate, minor unknowns, 1-2 days |
| 5 | Significant, some unknowns, 2-3 days |
| 8 | Large, notable unknowns, 3-5 days |
| 13 | Very large, high uncertainty — consider splitting |
| 21 | Too large — MUST be split before estimation |

**Planning poker approach:** If multiple team members estimate, note outliers and discuss. Divergence reveals hidden assumptions or misunderstood scope.

## Step 5: Apply three-point estimation (for timeline commitments)

When management needs dates, don't give single-point estimates. Use:

**For reliable estimates (well-understood domain):**
```
Expected = (Best + 4×MostLikely + Worst) / 6
```

**For less reliable estimates (new domain/technology):**
```
Expected = (Best + 3×MostLikely + 2×Worst) / 6
```

Present as ranges: "We'll likely finish in 3-4 weeks, with a tail risk of 6 weeks if [specific risk] materializes."

## Step 6: Capacity planning

**Team capacity is NOT 100% of available days.** Account for:

| Category | Typical Allocation | Adjust For |
|----------|-------------------|-----------|
| Feature work | 50-60% | Sprint goals, new development |
| KTLO (keep the lights on) | 20-30% | Bug fixes, maintenance, tech debt |
| On-call / incidents | 5-15% | Rotation size, system stability |
| Vacation / holidays / sick | 10-15% | Season, team size |
| Meetings / reviews / planning | 5-10% | Org size, process overhead |

**Calculate available capacity:**
```
Available dev-days = Team size × Sprint days × Feature allocation %
```

Example: 4 devs × 10 days × 55% = 22 dev-days for feature work per sprint.

**Ask the user** what their team's KTLO/oncall budget is — it varies widely (some teams budget 40%+ for operational load).

## Step 7: Account for often-forgotten work

Engineers consistently underestimate because they scope only the "coding" portion. Ensure estimates include:

| Often Missed | Examples | Typical Addition |
|-------------|----------|-----------------|
| **Testing** | Unit tests, integration tests, E2E tests, test fixtures, CI setup | 20-40% of implementation |
| **Deployment** | IaC changes, Kubernetes manifests, environment configs, feature flags | 1-3 days per new service |
| **Observability** | Metrics, structured logging, dashboards, alerts, tracing integration | 1-2 days per component |
| **Tooling** | CLI tools, migration scripts, data backfill scripts, admin endpoints | Often surprises late |
| **On-call handbooks** | Runbooks, troubleshooting guides, escalation procedures, architecture diagrams | 1-2 days per new system |
| **Documentation** | API docs, README updates, architecture decision records | 0.5-1 day per feature |
| **Security** | Auth integration, input validation, secrets management, pen test fixes | 1-3 days depending on exposure |
| **Data migration** | Schema changes, backfills, backward compatibility, rollback scripts | Highly variable — spike first |

**Rule of thumb:** If the estimate only covers "writing the code," double it to account for everything else needed to ship to production safely.

## Step 8: Map points to calendar

```
Velocity = story points completed per sprint (historical, or estimate conservatively)
Sprints needed = Total points / Velocity
Calendar time = Sprints × Sprint length
```

If no historical velocity exists, assume 60-70% of theoretical capacity for a new team.

## Step 9: Identify estimation risks

Flag anything that increases the cone of uncertainty:
- No prior art in the codebase for this type of work
- Dependencies on other teams or external services
- Unclear requirements (unresolved open questions in PRD)
- New technology the team hasn't shipped with before
- Data migration or backward compatibility requirements

Each risk should have a mitigation or a spike (timeboxed investigation) to reduce uncertainty before committing to dates.

## Step 10: Report

Present:

### Feature-Level Estimate
| Feature | T-Shirt | Points | Confidence | Risks |
|---------|---------|--------|-----------|-------|
| | | | High/Med/Low | |

### Capacity and Timeline
- Team capacity: X dev-days per sprint (after KTLO/oncall/vacation)
- Total estimated points: N
- Estimated velocity: Y points/sprint
- **Timeline: A-B sprints (C-D weeks)**
- Confidence interval: Best case X weeks, worst case Y weeks

### Key Risks and Mitigations
For each risk, note impact on timeline if it materializes.

### Recommendations
- Tasks to spike first (reduce uncertainty before committing)
- Features to defer if timeline is too long (descope P1/P2 items)
- Parallel work streams possible (which features are independent?)

Report **DONE** with the estimate.
Suggest: `/ygs-spike` if risky unknowns need validation, or `/ygs-wbs` to break into detailed tasks with individual estimates.
