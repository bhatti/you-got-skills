# Specialist: Spec Gaps

Review scope: PRs where missing or ambiguous specifications caused confusion, rework, or incorrect implementation.

**Rule: Only report findings where the PR data actually contains evidence. Missing data = skip. No speculation.**

## Step 1: Check linked issues for acceptance criteria

For each PR in the pre-computed data, examine the linked issue (if any):

1. Look for acceptance criteria indicators:
   - Explicit labels: "AC:", "Acceptance Criteria", "Definition of Done"
   - Checkbox lists: `- [ ]` or `- [x]` patterns
   - BDD format: "Given", "When", "Then"
   - Structured requirements: numbered lists with testable conditions

2. Classify each PR:
   - **Has AC:** Issue contains any of the above indicators
   - **No AC:** Issue exists but lacks acceptance criteria
   - **No issue:** PR has no linked issue at all

Record the counts for the metrics dashboard.

## Step 2: Check reviewer comments for spec confusion

Scan human reviewer comments for keywords indicating the spec was unclear:

```
confusion_keywords: "unclear", "what's the expected behavior", "spec doesn't mention",
  "missing requirement", "not specified", "ambiguous", "what should happen when",
  "is this intentional", "where is this documented", "what's the acceptance criteria",
  "can you clarify", "I don't understand the requirement", "contradicts the spec"
```

For each match, record:
- The PR number and reviewer username
- The exact comment text (or relevant excerpt)
- Whether the author responded with clarification or changed the implementation

## Step 3: Check implementation comments for assumption indicators

Scan PR diff comments and commit messages for signs the author filled in spec gaps with assumptions:

```
assumption_keywords: "I assumed", "not specified so I", "spec gap", "TODO: clarify",
  "guessing this should", "my interpretation", "unclear requirement",
  "making a judgment call", "need to confirm with", "placeholder until spec"
```

Also check for tech-debt signals introduced by spec ambiguity — authors working around unclear specs often leave debt markers:

```
tech_debt_keywords: "TODO", "FIXME", "HACK", "XXX", "workaround", "tech debt",
  "technical debt", "deprecated", "temporary fix", "revisit this", "not ideal but"
```

When found in PR diff comments or commit messages alongside assumption keywords, escalate severity — this indicates the spec gap forced a suboptimal implementation.

## Step 4: Check for rework signals

For each PR, look for evidence that spec gaps caused rework:
- Follow-up PRs that modify the same files within 2 weeks and reference the original PR
- Force-pushes after review comments about spec confusion
- Reviewer comments like "this changed since last review", "the requirement shifted"
- Revert PRs that cite unclear requirements

## Severity thresholds

| Condition | Severity |
|-----------|----------|
| Spec gap led to revert or follow-up PR fixing the same feature | CRITICAL |
| Human reviewer explicitly flagged spec issue (confusion keyword match) | HIGH |
| No acceptance criteria on PR with >200 lines changed | MEDIUM |
| Implicit assumption only (author comment, no reviewer flag) | MEDIUM |
| No linked issue but PR is small (<50 lines) and straightforward | LOW |

## Finding format

```
#### [SPEC] <title> — PR #N, #M | Confidence: HIGH
**Evidence:** Issue PROJ-123 has no acceptance criteria. Reviewer `@alice` commented: "What's the expected behavior when X?"
**Impact:** Without clear AC, implementation diverged — follow-up PR #67 changed the same feature within 5 days.
**Recommendation:** Add AC template to issue creation workflow. Suggest `.claude/skills/gotchas/spec-template.md`.
```

## Aggregation

After processing all PRs, summarize:
- Total PRs with AC vs without AC (spec coverage %)
- Top 3 most common confusion keywords across all PRs
- Authors most frequently making assumptions (may indicate they work on under-specified features)
- Reviewers most frequently flagging spec issues (may indicate they are compensating for weak specs)
