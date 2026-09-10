# Specialist: Spec Gaps

Review scope: PRs where missing or ambiguous specifications caused confusion, rework, or incorrect implementation.

**Rule: Only report findings where the PR data actually contains evidence. Missing data = skip. No speculation.**

## Step 1: Check linked issues for acceptance criteria

For each PR in the pre-computed data, examine the linked issue (if any):

1. Check the `has_acceptance_criteria` field in the pre-computed data. This field uses **semantic detection** — it checks for:
   - Explicit headings: "Acceptance Criteria", "Definition of Done", "Requirements", "Expected Behavior", "Success Criteria"
   - Checkbox lists: `- [ ]` or `- [x]` patterns (2+ checkboxes)
   - BDD format: "Given", "When", "Then" (2+ BDD statements)
   - Numbered requirements with modal verbs: "must", "should", "shall" (2+ numbered items)

2. **When `has_acceptance_criteria` is false**, also read the `Issue description excerpt` field — the issue may describe requirements using different language (e.g., "The feature should...", "Users need to be able to...", "When X happens, Y should occur"). Use your judgment to determine if the issue effectively communicates testable requirements, even without a formal AC section.

## CRITICAL: Inaccessible Issues Are Not Findings

When `has_acceptance_criteria` shows `unknown (Jira access denied — restricted project)`:
- The AC status is **UNKNOWN** — the issue EXISTS but the audit bot could not read it (HTTP 401/403)
- Do **NOT** count these PRs as "AC missing"
- Do **NOT** include them in the AC coverage denominator or numerator
- Treat them as a separate category: **Access denied**
- Never report "X/N PRs missing AC" when many of those X are access-denied — that is a false positive

3. Classify each PR into one of four buckets:
   - **Has AC:** `has_acceptance_criteria` is true, OR issue description contains clear testable requirements
   - **No AC (confirmed):** Issue was readable but lacks testable requirements in any form
   - **Access denied:** `has_acceptance_criteria` is unknown — issue is restricted, no data available
   - **No issue:** PR has no linked issue at all

Spec coverage % = (Has AC) / (Has AC + No AC confirmed). Exclude access-denied and no-issue PRs from both numerator and denominator.

Record the counts for the metrics dashboard. **Accuracy matters** — do not report 0% AC coverage if issues contain clear requirements in non-standard formats, and never conflate access-denied with absent AC.

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

## Step 3b: Analyze acceptance criteria quality and coverage

When a linked issue DOES have acceptance criteria, evaluate their quality:

1. **Completeness check** — Does the AC cover:
   - Happy path behavior (primary scenario)
   - Error/edge cases (what happens when input is invalid, service is down, etc.)
   - Non-functional requirements (performance bounds, security constraints, accessibility)
   - Backward compatibility or migration considerations

2. **Testability check** — Each AC should be verifiable. Flag vague criteria:
   - "Should be fast" (no measurable threshold)
   - "Handle errors gracefully" (no specific error cases)
   - "Must be secure" (no specific threats or controls)
   - "Should work like X" (reference to undocumented behavior)

3. **Template-only AC detection** — Flag AC sections that appear structural but contain no real requirements:
   - All checkboxes unchecked with generic text: "Feature works as expected", "All tests pass", "Code reviewed"
   - AC section header present but body is empty or contains only boilerplate
   - Identical AC text copied from a previous issue in the same project (identical wording across multiple issues)
   - AC that merely restates the PR title: "The bug is fixed" or "The feature is implemented"
   
   Classify these as **Template-only** (not absent, but also not meaningful). Include the count in spec coverage metrics as a separate category.

4. **Cross-PR AC pattern gaps** — Common categories of missing AC that appear across multiple PRs:
   - No AC for error/failure scenarios (what happens when the service is down, input is invalid, etc.)
   - No AC for performance bounds (e.g., "should handle 1000 requests/second")
   - No AC for backward compatibility or migration path
   - No AC for accessibility or security constraints
   
   Flag when the same *category* of AC gap appears in 3+ PRs — this indicates a systemic gap in how the team writes issues, not an individual oversight.

5. **PR-to-AC alignment** — For each PR with AC:
   - Do the changed files and PR description address each AC item?
   - Are there AC items with no corresponding test in the PR?
   - Did the implementation add behavior NOT covered by any AC item (scope creep)?
   - Did reviewers raise issues about behavior that should have been in the AC but wasn't?

6. **Cross-PR pattern** — Look for repeated AC failures:
   - Same type of missing edge case across multiple issues (e.g., pagination, auth edge cases)
   - Same reviewer repeatedly asking about the same category of missing AC
   - Issues from the same project/epic consistently lacking AC in one area

Classify AC quality as:
- **Strong**: Covers happy path + edge cases + non-functional, all testable
- **Weak**: Only covers happy path, vague on edge cases
- **Absent**: No AC at all
- **Template-only**: Has AC section header but content is boilerplate, empty checkboxes, or generic placeholder text — present in form but not in substance

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
- Total PRs by bucket: Has AC / No AC (confirmed) / Access denied / No issue
- Spec coverage % = Has AC / (Has AC + No AC confirmed) — exclude access-denied and no-issue from both numerator and denominator
- Top 3 most common confusion keywords across all PRs
- Authors most frequently making assumptions (may indicate they work on under-specified features)
- Reviewers most frequently flagging spec issues (may indicate they are compensating for weak specs)
