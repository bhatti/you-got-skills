# Specialist: Industry Best Practices

Review scope: compare PR patterns against established industry research on code review effectiveness, PR hygiene, and shift-left quality practices.

**Rule: Only report findings where the PR data actually supports the observation. Benchmarks are reference points, not mandates. No speculation.**

---

## Step 1: PR size distribution

Research benchmark (Google, Microsoft studies): review quality drops significantly above 400 LOC. Defect density correlation peaks at 200-400 LOC; above 800 LOC, reviewers miss 50%+ of issues.

For each PR, compute total lines changed (additions + deletions). Build a distribution:

```
| Size bucket  | Count | % of total | Benchmark  |
|--------------|-------|------------|------------|
| <100 LOC     | N     | X%         | Ideal      |
| 100-400 LOC  | N     | X%         | Good       |
| 400-800 LOC  | N     | X%         | Warning    |
| >800 LOC     | N     | X%         | Risk zone  |
```

Flag PRs >800 LOC individually. Check if oversized PRs had more post-merge fixes.

## Step 2: Rubber-stamp reviews

A rubber-stamp review is an approval with zero substantive comments on a non-trivial PR.

Detection criteria:
- PR has >100 lines changed
- Reviewer approved without leaving any comments, OR
- Reviewer left only bot-generated or single-word comments ("LGTM", "looks good", "+1")

Compute rubber-stamp rate: rubber-stamp approvals / total approvals on PRs >100 LOC.

Benchmark: <10% is healthy, 10-25% is warning, >25% indicates review culture problems.

## Step 3: Test coverage in PRs

For each PR, check whether test files were modified alongside production code:

- PR modifies production code but no test files -> flag
- PR modifies test files only -> skip (test-only change)
- PR modifies both -> check ratio (test LOC / production LOC)

Benchmark: production code changes should include test changes in >70% of PRs. Test-to-production ratio below 0.3 suggests insufficient testing discipline.

Cross-reference with CI status: did PRs without test changes still pass CI? (May indicate low baseline coverage.)

## Step 4: Documentation updates with code changes

For each PR that introduces new APIs, configuration options, or user-facing features:
- Check if any documentation files were modified (README, docs/, CHANGELOG, API docs)
- Check for doc-related labels or checklist items in the PR template

Benchmark: PRs introducing new features should update docs in >50% of cases.

## Step 5: Rollback preparedness

For PRs that modify critical paths (auth, payments, data pipelines, infrastructure):
- Check for rollback instructions in PR description
- Check for feature flags or gradual rollout mentions
- Check for revert plan or "how to revert" section

Benchmark: high-risk PRs should have rollback plans in >80% of cases.

## Step 6: Pre-merge CI/CD gate compliance

Check for:
- PRs merged with failing or pending CI checks
- Required checks that were bypassed (admin merge)
- PRs merged without any CI run (missing check suites)

Benchmark: 0% of PRs should merge with failing required checks.

## Step 7: Pattern consistency and tech-debt accumulation

Check across the full PR set for signs of tech-debt accumulation and inconsistent implementation patterns:

1. **Tech-debt signals:** Count PRs introducing TODO/FIXME/HACK/XXX markers in diff. Track ratio: debt-introducing PRs / total PRs. Benchmark: >20% suggests systemic under-investment in cleanup.

2. **Pattern divergence:** Look for PRs where reviewers flagged inconsistency:
   - "We already have X for this", "use the existing Y", "inconsistent with Z"
   - Multiple PRs solving the same problem differently (e.g., 3 different retry implementations, 2 config loading approaches)
   
3. **Cleanup ratio:** Count PRs that reduce tech debt (refactor, cleanup, remove deprecated code) vs those that add it. Benchmark: healthy repos have >15% cleanup PRs.

## Anti-patterns to flag

| Anti-pattern | Detection | Severity |
|--------------|-----------|----------|
| Oversized PRs (>800 LOC) that had post-merge fixes | Size + follow-up PR correlation | HIGH |
| Rubber-stamp reviews on complex PRs (>300 LOC, 3+ dirs) | Approval without comments on complex PR | HIGH |
| No tests with production changes (3+ consecutive PRs) | Test file absence pattern | MEDIUM |
| Missing changelogs for user-facing changes | No CHANGELOG/docs update on feature PRs | MEDIUM |
| No rollback plan for infrastructure changes | Missing revert instructions on infra PRs | MEDIUM |
| CI bypass patterns | Admin merge with failing checks | HIGH |
| Tech-debt accumulation (>20% PRs add TODO/FIXME) | Debt marker count in diffs | MEDIUM |
| Pattern divergence (3+ PRs with inconsistent approaches) | Reviewer "use existing X" comments | MEDIUM |
| Low cleanup ratio (<5% refactor/cleanup PRs) | PR title/label classification | LOW |

## Finding format

```
#### [PRACTICE] <title> — PR #N, #M | Confidence: HIGH
**Evidence:** 8 of 50 PRs (16%) exceeded 800 LOC. Of those, 3 had follow-up fix PRs within 1 week.
**Benchmark:** Google research: review effectiveness drops 50% above 400 LOC. Microsoft data: defect introduction rate doubles above 800 LOC.
**Impact:** Oversized PRs bypass effective review, leading to post-merge defects and rework.
**Recommendation:** Add PR size gate to CI (warn at 400 LOC, block at 800 LOC unless labeled `large-pr-approved`). Split large features using stacked PRs or feature flags.
```

## Aggregation

After processing all PRs, summarize:
- PR size distribution table
- Rubber-stamp rate with benchmark comparison
- Test-with-production rate
- Documentation update rate for feature PRs
- Rollback plan rate for high-risk PRs
- CI compliance rate
