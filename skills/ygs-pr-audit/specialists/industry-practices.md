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

## Step 2: Rubber-stamp and under-reviewed high-risk changes

A rubber-stamp is an approval where the approver left zero substantive feedback.

**Use pre-computed fields**: Each PR includes:
- `Substantive human comments` — count of human comments that are NOT rubber-stamp phrases
- `Rubber-stamp approvers` — list of approvers who left zero substantive comments on this PR
- `Author type: AI/bot-authored PR` — flag when the author's username ends with `bot` or matches known AI coding agents

Use these fields directly. Do not re-scan comment text manually.

**Rubber-stamp phrase list** — these phrases count as zero substantive feedback regardless of length:
- Explicit approvals: "LGTM", "L.G.T.M.", "looks good", "looks good to me", "approved"
- Short affirmations: "+1", "👍", "💯", "🚀", "✅", ":+1:", ":thumbsup:", ":ok_hand:"
- Generic praise: "nice work", "great", "perfect", "awesome", "excellent", "ship it", "merge it"
- Non-committal: "sounds good", "all good", "no issues", "no comments", "no concerns"
- Any emoji-only comment or comment under 60 characters matching the above patterns

**CRITICAL: Rubber-stamp on LOW-RISK changes is ACCEPTABLE — do NOT flag it.**
An LGTM on a cosmetic fix, doc update, or self-contained refactor is normal and expected.

Only flag rubber-stamp when ALL three conditions hold:
1. `Substantive human comments` = 0 (all approvers in `Rubber-stamp approvers`)
2. PR was approved and merged
3. Change has HIGH or MEDIUM blast-radius:

   **HIGH blast-radius → flag as HIGH finding:**
   - Security/auth/ACL — even a 2-line change to access-control or permission checks
   - Feature flags (`flags.yml`, `FeatureFlags.*`) — directly affects live production behavior
   - Billing, payments, or financial data pipelines
   - Query execution engines (ClickHouse, SQL query builders)
   - Data migrations or schema changes

   **MEDIUM blast-radius → flag as MEDIUM finding:**
   - Production configuration (`config/prod.*`, `.env.production`)
   - Infrastructure-as-code (`terraform/`, `k8s/`, `deploy/`, pipeline files)
   - API surface changes (routes, request/response schemas, public interface contracts)
   - SRE/monitoring/alerting configuration

   **LOW blast-radius → SKIP (rubber-stamp is fine):**
   - Internal refactors, docs, test-only changes, cosmetic fixes, style cleanup
   - LOC alone is NOT sufficient — a 1000-line refactor with no blast-radius signals = fine with LGTM

**IMPORTANT**: Never conflate rubber-stamp with "no human review":
- **No human review** = `Approved by` absent AND zero human comments → separate (usually more severe) finding
- **Rubber-stamp** = approved but no substantive feedback — reviewers present but not engaged

**Compute rubber-stamp rate**: (high-blast-radius PRs where ALL approvers are rubber-stamp) / (all high-blast-radius PRs).

Benchmark: <10% of high-blast-radius changes should have zero substantive review.

## Step 2b: Bot/AI-authored PR review adequacy

Bot-authored PRs (any PR whose author's username ends with `bot`, or known AI coding agents) carry **higher** correctness risk than human-authored ones because:
1. Bots optimize for implementing the stated spec — they miss emergent interactions
2. Bots don't ask "does this make sense" — they implement what was described
3. A silent approval provides zero evidence the reviewer understood what changed

**Detection**: Use the `Author type: AI/bot-authored PR` flag.

**Minimum acceptable review for bot-authored PRs**:
- ≥1 substantive human comment from a reviewer who can verify correctness — not just LGTM or approval
- The reviewer should state **what they validated** (e.g., "Verified the forecast threshold change: when no prediction data, the No forecast preview message is now shown correctly")
- Silent approval or rubber-stamp phrases on a bot-authored PR = MEDIUM finding
- Silent approval on a bot-authored PR touching HIGH blast-radius files = HIGH finding

**Positive pattern**: Bot PRs that received thorough review (≥3 substantive comments, AC-aligned) are worth calling out as examples of correct bot-human collaboration.

**Report metric**: "Bot-authored PRs with substantive review: X/N (N%)"

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

## Step 5: Rollback preparedness and SRE readiness

**High-risk change patterns** — trigger rollback check for ANY of these, regardless of directory:

| Pattern | File indicators |
|---------|----------------|
| Feature flags | `flags.yml`, `launchdarkly*`, `featureflags*`, `experiments*`, `toggles*` |
| Database migrations | `migrations/`, `*.migration.*`, `db/migrate/`, `flyway*`, `liquibase*` |
| Authentication/authorization | `auth*`, `rbac*`, `permission*`, `oauth*`, `saml*`, `iam*` |
| Infrastructure-as-code | `*.tf`, `*.tfvars`, `helm/`, `k8s/`, `kubernetes/`, `*.yaml` in infra dirs |
| API contract changes | OpenAPI/Swagger files, proto files, REST endpoint path changes |
| Configuration changes | `config/`, `*.env`, `settings*`, `application.yml` |

For each PR matching a high-risk pattern, check PR description AND review comments for:
```
rollback_keywords: "rollback", "revert", "feature flag", "gradual rollout", "canary",
  "blast radius", "what if this fails", "how to undo", "migration rollback", "dark launch"
```

**Known-limitation policy**: When a PR author documents a known limitation, deadlock risk, or "this breaks under condition X", check:
1. Did the author create a follow-up Jira/GitHub issue to track the limitation?
2. Is the limitation documented in AUTHZ.md, ARCHITECTURE.md, or a relevant `docs/` file?

If neither, report as `[PRACTICE]` MEDIUM — undocumented known limitations become invisible technical debt.

**SRE readiness signals** (check for absence):
- Observability: new code paths with no logging, metrics, or tracing
- Alerting: feature flag changes or threshold changes with no alert update
- Incident response: complex stateful changes with no runbook reference

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

## Step 8: Brittle test detection

For each PR that modifies test files, check for patterns that indicate fragile tests:

1. **Timing-based tests**: Look for `sleep`, `setTimeout`, `time.sleep`, `Thread.sleep`, fixed delay assertions, `waitFor` with short timeouts
2. **Excessive mocking**: Tests that mock more than 3 dependencies, or mock internal implementation details rather than interfaces
3. **Environment-dependent assertions**: Tests that depend on specific OS, timezone, locale, file paths, or network availability
4. **Flaky indicators**: Reviewer comments mentioning "flaky", "intermittent", "sometimes fails", "retry", or CI logs showing test retries
5. **Snapshot abuse**: Tests relying on large snapshots that change frequently (indicated by snapshot update commits)

Flag when 3+ PRs show brittle test patterns — this indicates systemic testing culture issues.

## Step 9: Human review analysis based on blast radius

Evaluate review quality relative to change risk:

1. **High blast-radius changes** (auth, payments, data pipelines, infra, config, feature flags):
   - Did these get proportionally more review scrutiny?
   - Were domain experts involved?
   - Were rollback plans discussed?

2. **Low-review risky changes**: Flag PRs modifying high-risk areas that received:
   - Zero human review comments
   - Only bot/automated comments
   - "LGTM" without substantive review

3. **Review depth vs risk mismatch**: Compare review comment count/quality against change risk level

## Step 10: Recurring reviewer correction patterns

Look for PRs where the same *type* of reviewer correction appears 3+ times across the PR set:

1. **Style rule corrections**: reviewer pointing out violations of documented rules (`.cursor/rules/`, `.claude/skills/`, `CONTRIBUTING.md`) — suggests those rules aren't being applied during implementation
2. **Repeated "use existing X"**: multiple PRs where reviewer says "we already have X for this" — suggests missing skill that auto-detects duplicate abstractions
3. **Test correctness corrections**: reviewer finding wrong expected values, missing edge cases, or incorrect mock setup — suggests test generation isn't being verified
4. **Branch/target corrections**: reviewer catching wrong base branch — suggests pre-push branch validation is missing

Report each pattern as `[PRACTICE]` with count of PRs affected and recommendation.

## Anti-patterns to flag

| Anti-pattern | Detection | Severity |
|--------------|-----------|----------|
| Oversized PRs (>800 LOC) that had post-merge fixes | Size + follow-up PR correlation | HIGH |
| Rubber-stamp reviews on high-risk changes (blast-radius areas: auth/flags/config/infra regardless of LOC) | Approved with zero substantive comments on high blast-radius PR | HIGH |
| No tests with production changes (3+ consecutive PRs) | Test file absence pattern | MEDIUM |
| Missing changelogs for user-facing changes | No CHANGELOG/docs update on feature PRs | MEDIUM |
| No rollback plan for infrastructure changes | Missing revert instructions on infra PRs | MEDIUM |
| CI bypass patterns | Admin merge with failing checks | HIGH |
| Tech-debt accumulation (>20% PRs add TODO/FIXME) | Debt marker count in diffs | MEDIUM |
| Pattern divergence (3+ PRs with inconsistent approaches) | Reviewer "use existing X" comments | MEDIUM |
| Low cleanup ratio (<5% refactor/cleanup PRs) | PR title/label classification | LOW |
| Brittle tests (timing, excessive mocking, env-dependent) | Test file analysis + reviewer comments | MEDIUM |
| High-risk change with no substantive review | Blast radius vs review depth mismatch | HIGH |
| Conflicting changes across PRs | Same files modified with divergent intent | MEDIUM |

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
