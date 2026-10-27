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
• <SprintName>: pending N pts (N) · in progress N pts (N) · review N pts (N) · done N pts (N) · total N pts (N) — ends <date>. WIP: OK | HIGH.
• Not pointed: <url|KEY>, <url|KEY> (only when some issues have no points; they are named, and add nothing to the totals)

*Call to Action*
<2-3 sentences: plain English summary of what needs attention RIGHT NOW. Be specific about PR numbers, issue keys, and who can help. Patterns:>
<"We have N PRs that need help — [list with context]. If you can review or resolve tasks on any of these today, it'll clear the backlog.">
<"KEY (summary, priority) is unassigned — who's closest to this area and has capacity?">
<"We're running high on WIP. Before picking up new work, check if there's something in-flight you can finish or unblock first.">

*Per-person status*
• *Alice:* Working on <url|KEY> (summary) — <url|PR #NNN> open 28h, no review yet. Points: pending N · in progress N · review N · done N.
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

*Sprint health:* N of N pts done (N%), N pts in review, N pts in progress, N pts pending — N days remaining
```

## Rules

- Emoji severity: 🔴 HIGH risk, 🟡 MEDIUM, ℹ️ LOW
- Always include Jira links: `<url|PROJ-NNN>` in every issue reference (Slack mrkdwn hyperlink)
- Always include PR links: `<url|PR #NNN>` in every PR reference
- If posting to stdout only: write `PROJ-NNN (url)` and `PR #NNN (url)`
- Points come from the STORY POINTS block (computed by the tooling); never sum or estimate them. Lanes: pending (not started or blocked) / in progress / review / done. Sub-tasks are not counted. Unknown points are written "points unavailable", never 0.
- Sprint health only when sprint boundary data is available
- "No data" is valid output — never fabricate status from silence
- Keep each person's entry to 2-3 sentences max
- Only include team members from the `team:` list in `.ygs/tracker.yml`
- Only include issues/PRs from the configured sprint board (board_id in tracker.yml)
- **NO hedging language** — never use "likely", "probably", "might be", "seems to be", "appears to". State observations as facts or omit them entirely. If data is ambiguous, say "no data" or describe what the data shows, not what it implies.
- **Slack signals are optional** — if `slack_messages` in signals.json is empty, omit all Slack references. Never write "No Slack activity detected" or ask people to post to the channel.
- **Surface priority and labels** — for Blocker/Critical/High priority issues, include the priority in the issue description (e.g., "working on PROJ-123 (High priority)"). For release-tracking labels (e.g., "2026-release"), include them when present. Skip Medium/Low/empty to reduce noise.
