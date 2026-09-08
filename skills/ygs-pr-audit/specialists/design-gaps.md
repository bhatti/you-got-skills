# Specialist: Design Gaps

Review scope: PRs where missing or insufficient design documentation caused architecture disagreements, review churn, or post-merge refactors.

**Rule: Only report findings where the PR data actually contains evidence. Missing data = skip. No speculation.**

## Step 1: Check linked issues for design doc references

For each PR in the pre-computed data, examine the linked issue and PR description for design documentation:

1. Look for design doc indicators:
   - Links to Confluence, Google Docs, Notion, or similar
   - References to ADRs (Architecture Decision Records)
   - References to RFCs or tech specs
   - Keywords: "design doc", "tech spec", "architecture doc", "RFC", "ADR"
   - Inline design sections in the PR description (headers like "## Design", "## Approach", "## Architecture")

2. Classify each PR by complexity:
   - **Complex:** >300 lines changed, touches 3+ directories, or introduces new abstractions
   - **Moderate:** 100-300 lines changed, touches 1-2 directories
   - **Simple:** <100 lines changed, single-purpose change

3. Flag complex PRs without design documentation.

## Step 2: Check PR comments for architecture disagreements

Scan human reviewer comments for keywords indicating design-level concerns:

```
design_keywords: "this should be in X layer", "wrong abstraction", "tight coupling",
  "circular dependency", "should we split this", "design doc says", "over-engineered",
  "under-abstracted", "violates separation of concerns", "this doesn't belong here",
  "alternative approach", "have we considered", "this will be hard to change later",
  "breaks the existing pattern", "inconsistent with how we do X"
```

For each match, record:
- The PR number and reviewer username
- The exact comment text
- Whether the comment led to significant code changes (force-push after the comment)

## Step 3: Check for review churn from design disagreements

For each PR, count the number of review rounds (approval cycles). Look for:
- 3+ review rounds on a single PR (suggests design was not agreed upon before coding)
- Large diff changes between review rounds (not just style fixes — structural changes)
- Comments like "let's take this offline", "need to discuss architecture", "sync needed"

## Step 4: Check for post-merge design rework

For each PR, look for follow-up PRs that:
- Modify the same core files within 2 weeks
- Have titles or descriptions referencing the original PR
- Contain refactoring keywords: "refactor", "restructure", "move to", "extract", "split out"
- Were authored by a different person (suggests the original design did not match team expectations)

## Step 5: Cross-reference with codebase patterns

If the repository has documented architecture patterns (in `ARCHITECTURE.md`, `docs/architecture.md`, `.claude/skills/architecture.md`, or similar):
- Check whether complex PRs align with documented patterns
- Flag deviations that were not called out by reviewers (blind spots)

## Step 6: Check for simplicity violations and pattern divergence

Scan PR comments and diff for signs of unnecessary complexity or inconsistent patterns:

```
simplicity_keywords: "over-engineered", "too complex", "simpler approach", "unnecessary abstraction",
  "premature generalization", "YAGNI", "do we really need", "why not just", "overkill",
  "this adds indirection without benefit", "too many layers"
```

Also check for pattern divergence — PRs that introduce a different approach to a concern already solved elsewhere in the codebase:
- Reviewer comments like "we already have X for this", "inconsistent with how we do Y", "why not use the existing Z"
- Multiple implementation patterns for the same concern (e.g., 2+ HTTP client wrappers, inconsistent error handling)
- New abstractions that duplicate existing ones

Flag when 3+ PRs introduce divergent patterns for the same concern — this indicates missing documentation of the canonical approach.

## Step 7: Check for conflicting changes and duplicate abstractions

Across the full PR set, look for:
1. **Conflicting changes**: Two or more PRs that modify the same files or modules with divergent intent (e.g., PR #A adds a caching layer while PR #B removes caching from the same module). Check file overlap in `file_paths`.
2. **Duplicate abstractions**: PRs that introduce new abstractions overlapping with existing ones (e.g., two different retry mechanisms, two config loading approaches, two HTTP client wrappers). Look for reviewer comments like "we already have X for this" or "use the existing Y".
3. **Inconsistent API patterns**: PRs that introduce different conventions for the same concern (error handling, pagination, auth) — indicated by reviewer comments about inconsistency.

## Severity thresholds

**IMPORTANT: Only raise design gap findings when there is actual evidence** — reviewer comments showing architecture debate, explicit design questions, OR the PR is part of a large feature/epic (>500 LOC touching multiple subsystems). A large PR without a design doc is NOT automatically a design gap unless reviewers raised concerns or it touches architectural boundaries.

| Condition | Severity |
|-----------|----------|
| Design gap led to 3+ review rounds AND post-merge refactor | CRITICAL |
| Reviewer comments show explicit architecture debate (design_keywords match) | HIGH |
| Design doc existed but did not cover the implementation choice made | HIGH |
| Large feature (>500 LOC, 3+ dirs) without design doc AND reviewer raised concerns | MEDIUM |
| Conflicting changes between PRs modifying same module | MEDIUM |
| Architecture disagreement in comments but resolved in 1 round | MEDIUM |
| No design doc for complex feature but no reviewer concerns | LOW |
| No design doc for moderate feature, no reviewer concerns | LOW |

## Finding format

```
#### [DESIGN] <title> — PR #N | Confidence: HIGH
**Evidence:** PR #45 added 520 lines across 4 directories with no design doc reference. Reviewer `@bob` commented: "This should be in the service layer, not the controller." PR went through 4 review rounds.
**Impact:** Design disagreements in-PR cause review fatigue and delay merges. Follow-up PR #52 refactored the same code 8 days later.
**Recommendation:** Require design doc link for PRs >300 LOC. Add lightweight ADR template to `.claude/skills/gotchas/design-template.md`.
```

## Aggregation

After processing all PRs, summarize:
- Complex PRs with design docs vs without (design coverage %)
- Average review rounds for PRs with design docs vs without
- Top 3 architecture disagreement patterns across all PRs
- Time-to-merge difference between design-documented and undocumented complex PRs
