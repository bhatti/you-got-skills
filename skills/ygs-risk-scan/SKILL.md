---
name: ygs-risk-scan
description: Ranked risk report for the current sprint — stale work, dependency chains,
  review bottlenecks, capacity gaps. Callable standalone or invoked by ygs-standup.
  Answers "what is about to go wrong?" not "what happened yesterday?"
---

# Risk Scan

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.

**Principle:** A risk not named before it blocks someone costs a sprint.

## Step 1: Init

Follow `~/.claude/skills/you-got-skills/skills/shared/init.md` (config load + credential verify).

## Step 2: Gather sprint state

Using queries from `shared/tracker.md`:
- All open issues in current sprint: status, assignee, last-updated
- Blocked issues (label or status)
- Open PRs: age and review activity
- Sprint end date and remaining days (tracker.md "Sprint end date and capacity" section)
- Issues not yet started (count for capacity check)
- Dependency links between issues

## Step 3: Build dependency graph

For each issue: does it block another? Is the blocker stale, blocked, or unassigned?
Are the two issues owned by different people (handoff risk)?

Flag any chain where a stale/blocked issue sits upstream of in-progress work.

## Step 4: Capacity check

```
remaining_days   = sprint_end_date - today          (from tracker sprint query)
issues_not_started = count of not-started sprint items
avg_days_per_issue = historical_velocity / sprint_days  (from tracker velocity query)
capacity_needed  = issues_not_started × avg_days_per_issue
capacity_available = team_size × remaining_days × (hours_per_day / 8)
```

If `capacity_needed > capacity_available`: flag HIGH risk with specific items to cut.
If sprint end date unavailable: note the gap and skip this step.

## Step 5: Rank and report

Apply `~/.claude/skills/you-got-skills/skills/shared/risk-criteria.md`.

```
RISK SCAN — Sprint N — {date}

🔴 HIGH (address today)
  1. PROJ-38 stale 4d — blocks PROJ-44 (alice) and PROJ-47 (charlie)
     Action: reassign or descope PROJ-38

🟡 MEDIUM (watch)
  2. PR #89 open 3d — bob sole reviewer, 2 others waiting
     Action: pair-review or reassign one PR

ℹ️ LOW
  3. PROJ-52 not started, 3d remaining — confirm fits sprint

Sprint: 8/14 done (57%), 4 days left
Velocity needed: 1.5/day (historical avg: 1.2/day)
```

## Step 6: Recommended actions

For each HIGH/MEDIUM: one concrete action — reassign, descope, unblock, or escalate.

Exit JSON for Formicary:
```json
{"status": "DONE", "high_risks": N, "medium_risks": N, "sprint_health_pct": N}
```

## Invocation modes

- `/ygs-risk-scan` — full sprint risk report
- Called by `ygs-standup` — returns structured risk list for inclusion in brief
- Formicary weekly job (Monday before sprint planning)
