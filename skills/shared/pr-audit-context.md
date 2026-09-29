# PR Audit Context — Shared Reference

Common definitions for pr-audit specialists. All specialists reference this file for consistent classification.

---

## PR data format

The pre-computed PR data provides these fields per PR:

```json
{
  "number": 123,
  "title": "Add user authentication endpoint",
  "author": "username",
  "merged_at": "2025-01-15T10:30:00Z",
  "jira_summary": "[optional] Jira issue summary when title is a bare key like PROJ-123",
  "files_changed": 12,
  "additions": 350,
  "deletions": 45,
  "linked_issue": {
    "key": "PROJ-456",
    "title": "Implement login flow",
    "body": "...",
    "labels": ["feature", "auth"],
    "has_acceptance_criteria": true
  },
  "bot_comments": [
    {"author": "github-actions[bot]", "body": "...", "category": "ci"}
  ],
  "human_comments": [
    {"author": "alice", "body": "...", "type": "review_comment"}
  ],
  "review_decision": "APPROVED",
  "review_rounds": 2,
  "ci_status": "pass",
  "required_checks_passed": true
}
```

Not all fields are present for every PR. Handle missing fields gracefully — do not infer from absence.

**Canonical `ci_status` values** (normalized from GH and Bitbucket by pr_metadata):
- `"pass"` — all checks passed (GH: success; BB: SUCCESSFUL)
- `"fail"` — one or more checks failed (GH: failure/error; BB: FAILED)
- `"pending"` — checks still running
- `"unknown"` — CI data unavailable (common for BB open PRs)

**Canonical `state` values:**
- `"merged"` — PR was merged into the target branch
- `"open"` — PR is currently open / in review
- `"declined"` — PR was declined or closed without merging

**Canonical `pr_type` values** (classified by `classify_pr_type` from title + jira_summary + labels):
- `"feature"` — new functionality or unclassifiable change (default fallback; most generic)
- `"bug"` — defect fix, revert, regression, crash
- `"refactor"` — code cleanup, restructure, rename, extract
- `"chore"` — dependency bump, version update, CI/build change
- `"security"` — vulnerability fix, CVE, exploit mitigation
- `"test"` — test additions or fixes (checked before bug; "[Flaky Test]" → test)
- `"docs"` — documentation, README, changelog

Note: `"unknown"` is no longer emitted. Jira-linked PRs use `jira_summary` for classification when the PR title is a bare Jira key (e.g. `"CRIBL-44875"`).

---

## Bot detection rules

A reviewer is classified as a **bot** if ANY of:
1. Username ends with `bot` or `[bot]` (case-insensitive match)
2. Username is in this known set:
   - `github-actions[bot]`
   - `dependabot[bot]`
   - `renovate[bot]`
   - `codecov[bot]`
   - `sonarcloud[bot]`
   - `snyk-bot`
   - `codeclimate[bot]`
   - `netlify[bot]`
   - `vercel[bot]`
3. Comment body contains structured skill invocation markers (JSON output blocks, `ygs-*` skill names, `::add-task-context` markers)

Everything else is **human**. When in doubt, classify as human (conservative — avoids undercounting human effort).

---

## Issue-linking conventions

PRs may reference issues through:
- **Jira keys:** regex `[A-Z][A-Z0-9]+-\d+` (e.g., `PROJ-123`, `ENG-456`)
- **GitHub issue refs:** `#\d+` in PR body or title
- **Closing keywords:** `Closes #N`, `Fixes #N`, `Resolves #N` (case-insensitive)
- **Jira closing keywords:** `Closes PROJ-123`, `Fixes ENG-456`

A PR may reference multiple issues. Use the first linked issue as the primary for spec-gap analysis.

---

## Skill-to-bot mapping

To determine which skill (if any) produced a bot comment:
1. **Explicit markers:** Look for skill names in the comment body (`ygs-review-pr`, `ygs-security-review`, `ygs-code-review`)
2. **JSON status lines:** Comments ending with `{"status":"DONE",...}` are skill outputs
3. **Structured headers:** Comments with `## Code Review` or `## Security Review` headers followed by findings in the standard format
4. **Inference from content:** If a bot comment discusses security topics, map it to the security review skill; if it discusses code quality, map it to the code review skill

When mapping is ambiguous, record the bot comment category but do not attribute it to a specific skill.
