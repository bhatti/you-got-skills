# Specialist: Skills Gaps (Bot vs Human Review Delta)

Review scope: identify categories of issues that human reviewers catch but automated tooling (bots, CI, skills) misses. This is the most actionable dimension — every gap found here maps directly to a skill improvement.

**Rule: Only report findings where the PR data actually contains evidence. Missing data = skip. No speculation.**

---

## Step 1: Separate bot vs human comments

Using the pre-classified data in the PR context, verify the classification:

**Bot detection rules:**
- Username ends with `bot` or `[bot]` (case-insensitive)
- Username is in known set: `github-actions[bot]`, `dependabot[bot]`, `renovate[bot]`, `codecov[bot]`, `sonarcloud[bot]`, `snyk-bot`, `codeclimate[bot]`, `netlify[bot]`, `vercel[bot]`
- Comment body contains skill invocation markers (e.g., `ygs-review-pr`, `ygs-security-review`, audit report JSON)

**Human:** Everything else. Verify that classified-human reviewers are not bots by checking if their comments look templated or automated.

Record per-PR:
- Count of bot comments
- Count of human comments
- List of unique bot identifiers
- List of unique human reviewers

## Step 2: Categorize human review comments by type

For each human review comment, classify it into one or more categories using keyword matching:

```
correctness:    "bug", "wrong", "incorrect", "breaks", "fails", "regression",
                "crash", "null", "undefined", "off-by-one", "infinite loop",
                "race condition", "deadlock"

security:       "injection", "auth", "token", "credential", "xss", "csrf",
                "sanitize", "escape", "privilege", "permission", "secret",
                "vulnerability", "exposure"

performance:    "slow", "memory", "O(n", "cache", "timeout", "leak",
                "unbounded", "N+1", "batch", "pagination", "index"

architecture:   "coupling", "abstraction", "interface", "dependency", "layer",
                "separation of concerns", "single responsibility", "pattern",
                "doesn't belong", "wrong module"

testing:        "test", "coverage", "edge case", "regression test", "mock",
                "assertion", "missing test", "untested", "flaky"

documentation:  "doc", "comment", "unclear", "confusing", "readme",
                "changelog", "jsdoc", "docstring", "explain"
```

If a comment matches multiple categories, assign it to all matching categories.

## Step 3: Categorize bot comments by type

Apply the same category classification to bot comments. This lets us compute the overlap.

## Step 3b: Account for bot internal self-review

AI coding agents (Claude Code, Copilot, Cursor, etc.) perform internal self-review before submitting code — including security scanning, correctness checks, and style enforcement. This self-review is NOT visible in PR comments because it happens before the code is committed. Therefore:

- **Do NOT claim that a bot "missed" a finding category just because no bot PR comment exists for that category.** The bot may have caught and fixed similar issues during implementation, before the PR was created.
- **Only flag a skill gap as "bot missed" when a human reviewer found an actual defect in the bot-authored code** — meaning the bot's internal self-review failed to catch that specific issue.
- **Bot-authored PRs with zero external bot review comments are normal** — the bot's quality gate is internal, not expressed as PR comments.
- **Focus on what actually slipped through**: if a human reviewer found a real bug, security issue, or design problem in bot-authored code, THAT is the genuine skill gap worth reporting.

When reporting findings, distinguish between:
- **Genuine skill gap**: Human found a real defect in bot-authored code (the bot's internal review missed it)
- **Process gap**: No external automated review tool ran on the PR (CI/SAST not configured)
- **Coverage gap**: Bot's internal review covers some categories but not others (evidenced by patterns of human catches)

## Step 4: Compute set difference (the gap)

For each PR and each category:
1. Did any human reviewer flag an issue in this category? (human_found = true/false)
2. Did any bot comment in the same PR cover this category? (bot_found = true/false)
3. **Gap = human_found AND NOT bot_found** — the human caught something the bot missed.
4. **If the PR author is a bot**: a human-only finding in this PR is a genuine self-review gap (higher signal)
5. **If the PR author is human**: a human-only finding means external review tooling is missing (lower signal — the bot wasn't involved)

Build a gap matrix:

```
| PR    | correctness | security | performance | architecture | testing | documentation |
|-------|-------------|----------|-------------|--------------|---------|---------------|
| #45   | human-only  | both     | —           | human-only   | bot-only| —             |
| #67   | —           | human-only| —          | human-only   | both    | human-only    |
```

Cells marked `human-only` are the skill gaps.

## Step 5: Cross-reference with installed skills

Check which review skills are installed:

```bash
ls ~/.claude/skills/you-got-skills/skills/ygs-review-*/SKILL.md 2>/dev/null
ls ~/.claude/skills/you-got-skills/skills/ygs-security-review/SKILL.md 2>/dev/null
ls ~/.claude/skills/you-got-skills/skills/ygs-code-review/SKILL.md 2>/dev/null
ls .claude/skills/*/SKILL.md 2>/dev/null
```

For each installed skill, note which categories it covers. Map gaps to specific skill files.

## Step 6: For each gap, recommend specific skill update

For every `human-only` cell in the gap matrix that appears in 2+ PRs:

1. Identify which skill should have caught it (or recommend creating a new one)
2. Identify the specific file and section within the skill
3. Describe what detection pattern to add (keywords, code patterns, heuristics)
4. If no skill exists for the category, recommend creating a repo-local skill under `.claude/skills/`

## Severity thresholds

| Condition | Severity |
|-----------|----------|
| Same gap category across 3+ PRs (systemic blind spot) | CRITICAL |
| Security or correctness gap missed by all bots | HIGH |
| Architecture or performance gap missed by all bots | MEDIUM |
| Testing gap (human noted missing tests, bot did not) | MEDIUM |
| Style or documentation gap only | LOW |

## Finding format

```
#### [SKILL-GAP] <title> — PR #N, #M, #K | Confidence: HIGH
**Evidence:** Human reviewer `@alice` flagged SQL injection risk in PRs #45, #67, #89. Bot review (`ygs-security-review`) ran on all three PRs but did not flag this pattern.
**Gap matrix:** security category — human-only in 3/50 PRs (6%)
**Skill gap:** `ygs-security-review/SKILL.md` checks for `execute()` but not f-string interpolation in Python SQL queries.
**Recommendation:** Add f-string SQL detection to `ygs-security-review/specialists/injection.md`:
  - Pattern: `f"SELECT|INSERT|UPDATE|DELETE.*{` in Python files
  - Or create repo-local `.claude/skills/security-review/sql-injection.md` with project-specific ORM patterns.
```

## Aggregation

After processing all PRs, compute:

**Per-category gap counts:**
- correctness: N human-only findings across M PRs
- security: N human-only findings across M PRs
- (repeat for all categories)

**Skill coverage heatmap:**

```
| Category      | Bot catches | Human-only | Gap rate |
|---------------|------------|------------|----------|
| correctness   | N          | N          | X%       |
| security      | N          | N          | X%       |
| performance   | N          | N          | X%       |
| architecture  | N          | N          | X%       |
| testing       | N          | N          | X%       |
| documentation | N          | N          | X%       |
```

**Top skill improvement priorities** (ranked by gap rate * severity):
1. Highest-gap category with specific skill file and section to update
2. Second-highest
3. Third-highest

**Reviewer load analysis:**
- Which human reviewers are doing the most gap-filling work?
- Are certain reviewers specialized (e.g., one person catches all security issues)?
- Would upskilling bots in their specialty areas reduce their review burden?
