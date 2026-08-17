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

*Board status*
• <SprintName>: N total, N done, N in-progress/review, N not started — ends <date>. WIP: OK | HIGH.

*Call to Action*
<2-3 sentences: plain English summary of what needs attention RIGHT NOW. Be specific about PR numbers, issue keys, and who can help. Patterns:>
<"We have N PRs that need help — [list with context]. If you can review or resolve tasks on any of these today, it'll clear the backlog.">
<"KEY (summary, priority) is unassigned — who's closest to this area and has capacity?">
<"We're running high on WIP. Before picking up new work, check if there's something in-flight you can finish or unblock first.">

*Per-person status*
• *Alice:* Working on <url|KEY> (summary) — <url|PR #NNN> open 28h, no review yet.
• *Bob:* No tracker activity since Monday (3 days). No standup channel message.
• *Charlie:* Merged <url|KEY>. Two PRs in review.

*Risks*
🔴 <url|KEY> stale Nd — blocks <url|KEY> (person) and <url|KEY> (person)
🟡 <url|KEY> shows "In Review" but PR is still draft — update board to reflect reality
🟡 <url|PR #NNN> open Nd, no reviewers — close or assign? (N lines, N files)

*Discussion (bring to the meeting)*
1. <url|KEY> stale Nd blocks two people — descope or reassign?
2. Review load: Bob has N pending items — redistribute?
3. <url|PR #NNN> open Nd with no reviewers — close it, or does someone have 15 min?

*Sprint health:* N/N issues done (N%), N in progress, N not started — N days remaining
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
