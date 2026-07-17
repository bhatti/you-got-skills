---
name: ygs-sprint-plan
description: AI-assisted sprint planning — given backlog + team capacity, propose what
  to pull in, flag dependencies and blockers before they enter the sprint, surface
  items that need design decisions before implementation can start.
---

# Sprint Plan

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.

**Principle:** Problems caught before sprint planning are free. Problems caught mid-sprint cost the whole team.

## Step 1: Init

Follow `~/.claude/skills/you-got-skills/skills/shared/init.md` (config load + credential verify).

Ask user if not in config:
- Any team members unavailable (PTO, part-time)?
- Any committed deliverables (release date, customer deadline)?

## Step 2: Pull current state

Using queries from `shared/tracker.md`:
- Backlog candidates (open issues not in any sprint)
- Carry-over: open issues from last sprint/milestone
- Last 2 sprint velocity (issues/points completed)
- Resolve numeric sprint ID for the new sprint (tracker.md "Resolve JIRA sprint ID" section)

## Step 3: Calculate capacity

```
available_person_days = team_size × sprint_days × (1 - meeting_overhead_pct/100)
  → subtract PTO days for named individuals
carry_over_cost       = estimated days for open carry-over items
net_capacity          = available_person_days - carry_over_cost
avg_issue_size        = last_sprint_points / last_sprint_count  (or ask user)
target_count          = floor(net_capacity / avg_issue_size)
```

## Step 4: Dependency and readiness check

For each backlog candidate:
1. Open dependency (blocked by unresolved issue)?
2. Missing acceptance criteria? (look for "TBD", open questions in body)
3. Requires design decision not yet made?

Flag items failing any check as NOT recommended — do not pull them in.

## Step 5: Propose sprint scope

Priority order: carry-over → committed → high-priority no blockers → high-priority resolvable early → medium fitting capacity.

```
PROPOSED SPRINT N SCOPE

Carry-over (must)
  ✓ PROJ-38 — Auth fix (est. 2d) — alice
  ✓ PROJ-41 — DB migration (est. 1d) — bob

New items
  ✓ PROJ-52 — Rate limiter (est. 3d) — charlie
  ✓ PROJ-55 — Metrics dashboard (est. 2d) — alice — needs PROJ-52 first ⚠️

Total: 8 person-days  |  Capacity: 32  |  Slack: 24d (healthy)

NOT recommended
  ✗ PROJ-60 — Blocked by PROJ-38 (carry-over)
  ✗ PROJ-63 — Missing acceptance criteria
  ✗ PROJ-71 — Open arch decision in issue body
```

## Step 6: Groom flagged items

For each NOT recommended:
- Missing AC → draft criteria, ask user to approve
- Open design question → surface it, ask if resolvable now
- Dependency → confirm whether blocker can also be pulled in

## Step 7: Confirm and apply

On user confirmation, assign issues to sprint using the numeric sprint ID from Step 2:

```bash
# JIRA — use numeric sprint ID (name string not accepted)
SPRINT_ID=<from tracker.md "Resolve JIRA sprint ID" query>
acli -p jira jira issue update PROJ-XX --sprint "$SPRINT_ID"
# or via REST:
curl -sf -X POST "$JIRA_BASE_URL/rest/agile/1.0/sprint/$SPRINT_ID/issue" \
  -H "Authorization: Basic $JIRA_AUTH" -H "Content-Type: application/json" \
  -d '{"issues": ["PROJ-XX"]}'

# GitHub — assign to milestone
gh issue edit NUMBER --milestone "Sprint N"
```

Report **DONE**: scope list, capacity utilization %, deferred items and why, items needing grooming.

Exit JSON for Formicary:
```json
{"status": "DONE", "issues_planned": N, "capacity_pct": N, "deferred": N, "needs_grooming": N}
```
