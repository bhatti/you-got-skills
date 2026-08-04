# Standup Brief Format

Read `~/.claude/skills/you-got-skills/skills/shared/output-format.md` for shared row templates and enforcement rules.

## STRICT RULES (violation = wrong answer)

1. Every Jira issue MUST use `<url|PROJ-NNN>` link format (Slack) or `PROJ-NNN (url)` (stdout)
2. Every PR MUST use `<url|PR #NNN>` link format (Slack) or `PR #NNN (url)` (stdout)
3. Never output status for people not in the team list from `.ygs/tracker.yml`
4. Never fabricate — "No data" is the correct output when there is no signal
5. Filter every issue/PR to the configured sprint + team (board_id in tracker.yml)
6. NEVER hardcode org names, project keys, usernames, or URLs — use values from config/signals

## Output structure

```
📋 *Standup Brief — {date} {time}*

*Per-person status*
• *Alice:* Closed <url|PROJ-NNN> (auth fix). Working on <url|PROJ-NNN> (rate limiter) — <url|PR #NNN> open 28h, no review yet.
• *Bob:* No tracker activity since Monday (3 days). No standup channel message.
• *Charlie:* Merged <url|PROJ-NNN>. Two PRs in review.

*Risks*
🔴 <url|PROJ-NNN> stale 4d — blocks <url|PROJ-NNN> (alice) and <url|PROJ-NNN> (charlie)
🟡 <url|PR #NNN> open 3d — bob sole reviewer, 2 others waiting

*Discussion (bring to the meeting)*
1. <url|PROJ-NNN> stale 4d blocks two people — descope or reassign?
2. Review load: Bob has 3 pending items — redistribute?

*Sprint health:* 8/14 issues done (57%), 3 in progress, 3 not started — 4 days remaining
```

## Rules

- Emoji severity: 🔴 HIGH risk, 🟡 MEDIUM, ℹ️ LOW
- Always include Jira links: `<url|PROJ-NNN>` in every issue reference (Slack mrkdwn hyperlink)
- Always include PR links: `<url|PR #NNN>` in every PR reference
- If posting to stdout only: write `PROJ-NNN (url)` and `PR #NNN (url)`
- Sprint health only when sprint boundary data is available
- "No data" is valid output — never fabricate status from silence
- Keep each person's entry to 2-3 sentences max
- Only include team members from the `team:` list in `.ygs/tracker.yml`
- Only include issues/PRs from the configured sprint board (board_id in tracker.yml)
