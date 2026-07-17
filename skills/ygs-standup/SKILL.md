---
name: ygs-standup
description: Daily standup brief — gather signals from tracker (GitHub/JIRA) and Slack,
  synthesize per-person status, flag silence and dependency risks, surface discussion
  questions. Replaces person-by-person status recitation with evidence-backed narrative.
---

# Standup Brief

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.

**Principle:** The meeting is not where you collect information — it's where you make decisions.

## Step 1: Init

Follow `~/.claude/skills/you-got-skills/skills/shared/init.md` (config load + credential verify + Slack check).

## Step 2: Gather signals (run in parallel — failures degrade gracefully)

Using queries from `shared/tracker.md`:
- Issues updated last 24h per team member
- Issues with no activity > 2 days (silence signal)
- Open PRs: age, review status, no reviewer activity > 1 day
- Blocked issues (label or status)

Using queries from `shared/slack.md`:
- Blocker/help/stuck keywords in standup channel, last 24h
- Team members with no standup channel message in 2+ days

Cross-signals (derive after both):
- Issue B blocked on stale Issue A owned by someone else → dependency risk
- PR open > 2 days with no review → identify bottleneck reviewer

## Step 3: Synthesize per-person

For each team member (2-3 sentences, evidence-backed):

```
**Alice:** Closed PROJ-42 (auth fix). Working on PROJ-51 (rate limiter) — PR open
28h, no review yet. [Slack: "waiting on infra cert renewal"]
```

- Every claim traces to a ticket, PR, or Slack message
- No data → "No tracker activity in last 24h" (never fabricate)
- Silence → "No updates since Monday (3 days)"

## Step 4: Identify risks

Apply `~/.claude/skills/you-got-skills/skills/shared/risk-criteria.md`. Produce ranked list:

```
🔴 PROJ-51 PR no reviewer — bob is only approver, unresponsive (Slack)
🟡 PROJ-38 stale 4 days — blocks PROJ-44 (alice) and PROJ-47 (charlie)
```

## Step 5: Generate discussion questions

2-3 items requiring human judgment — NOT status recitation:

```
1. PROJ-38 stale 4d blocks two people — descope or reassign?
2. Bob has 3 PRs queued — redistribute review load?
```

## Step 6: Deliver

Format using `references/brief-format.md`. Post to Slack if `SLACK_BOT_TOKEN` set. Always print to stdout.

Exit JSON for Formicary:
```json
{"status": "DELIVERED", "risk_count": N, "discussion_questions": N, "silence_count": N}
```

## Invocation modes

- `/ygs-standup` — full brief now
- `/ygs-standup @alice` — single person
- Formicary cron: reads `.ygs/tracker.yml`, posts brief, exits with JSON
