# Shared Output Format

Consistent bullet-row format used by ygs-standup, ygs-risk-scan, ygs-pr-queue, and jira-query.

## OUTPUT FORMAT: BULLET ROWS, NOT MARKDOWN TABLES

**CRITICAL:** Output MUST use `•`-prefixed bullet rows. Do NOT output markdown tables (`| col | col |`).
Markdown tables are stripped or mangled by Slack; bullet rows render correctly.

Correct format:
```
• <url|KEY> [Type] Summary — _Assignee_ · Status · 3d old
```

Wrong format (do NOT use):
```
| KEY | Summary | Assignee | Status |
|-----|---------|----------|--------|
```

## STRICT ENFORCEMENT RULES (violation = wrong answer)

1. Every Jira issue line MUST include a clickable Slack link: `<url|PROJ-NNN>`
2. Every PR line MUST include a clickable Slack link: `<url|PR #NNN>`
3. If posting to stdout (not Slack): write `PROJ-NNN (url)` and `PR #NNN (url)`
4. NO fabricated data — if a field is unknown write `—`
5. Filter to configured team only — never show issues/PRs from other teams or boards
6. Emoji severity: 🔴 HIGH · 🟡 MEDIUM · ℹ️ LOW
7. NEVER hardcode org names, project keys, usernames, or URLs — derive from config/env vars

## Issue row format

```
• <jira_url|PROJ-NNN> [Type] Summary — _Assignee_ · Status · Priority · Nd old
```

Example (use values from config, not literal project names):
```
• <https://yourorg.atlassian.net/browse/PROJ-1234|PROJ-1234> [Bug] Short summary — _alice_ · In Progress · High · 3d old
```

## PR row format

```
• <pr_url|PR #NNN> Summary — _author_ · approved-by: @Name | pending: @Name · Nd old
```

Example:
```
• <https://github.com/yourorg/yourrepo/pull/5678|PR #5678> Short summary — _bob_ · pending: @alice · 2d old
```

## Section headers

ALL CAPS on their own line, no markdown decorators:

```
OPEN ISSUES — SPRINT N
OPEN PRS
RISKS
DISCUSSION
```

## Table ordering

1. By severity (HIGH → MEDIUM → LOW), then
2. Within severity: by age descending (oldest first)

## Team filter (MANDATORY)

Before outputting any item, verify it belongs to the configured team:
- Jira: assignee must be in the `team:` list from `.ygs/tracker.yml`  
  OR the issue must be in `sprint in openSprints()` for the configured `board_id`
- GitHub/Bitbucket: PR author or assignee must be in the `team:` list
- NEVER output items assigned to people not in the team list
- If `team:` is empty in config, skip the filter but warn: `[output] team list empty — showing all assignees`
