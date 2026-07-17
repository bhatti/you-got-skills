# Standup Brief Format

## Output structure

```
📋 *Standup Brief — {date} {time}*

*Per-person status*
• *Alice:* Closed #{id} (auth fix). Working on #{id} (rate limiter) — PR open 28h, 
  no review yet. Blocker: waiting on infra cert (from Slack).
• *Bob:* No tracker activity since Monday (3 days). No standup channel message.
• *Charlie:* Merged #{id}. Two new PRs in review. On track.

*Risks*
🔴 #{id} stale 4 days — blocks #{id} (alice) and #{id} (charlie)
🟡 Bob's PR queue: 3 items waiting, likely review bottleneck

*Discussion (bring to the meeting)*
1. #{id} stale 4d blocks two people — descope or reassign?
2. Review load distribution: Bob has 3 pending items — redistribute?

*Sprint health:* 8/14 issues done, 3 in progress, 3 not started — 4 days remaining
```

## Rules

- Emoji status indicators: 🔴 HIGH risk, 🟡 MEDIUM, ℹ️ LOW
- Link ticket IDs to tracker URLs when posting to Slack
- Sprint health only when sprint boundary data is available
- "No data" is valid output — never fabricate status from silence
- Keep each person's entry to 2-3 sentences max
