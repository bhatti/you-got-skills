---
name: ygs-pr-queue
description: Show open PRs for your sprint team — scoped to the boards and assignees
  from your active sprint. Identifies review bottlenecks, stale PRs, and PRs waiting
  for action. Use instead of a raw PR list when you want team-filtered context.
---

# PR Queue — Team Sprint View

**Principle:** Show PRs that matter to your sprint, not every open PR in the repo.

## Step 1: Init

Follow `~/.claude/skills/you-got-skills/skills/shared/init.md` (config load + credential verify).

## Step 2: Identify team and sprint

Using queries from `shared/tracker.md`:
- Load active sprint(s) for the configured project/board
- Extract team members = unique assignees with issues in the sprint
- If STANDUP_TEAM_MEMBERS is set, use that list to filter instead

## Step 3: Gather open PRs scoped to team

For GitHub (GH_ORG / GH_REPO):
- Fetch open PRs where author is in team members list
- Also fetch PRs where a team member is a requested reviewer

For Bitbucket (BITBUCKET_WORKSPACE / BITBUCKET_REPO):
- Fetch open PRs where author is in team members list
- Also fetch PRs where a team member is a reviewer

For each PR collect:
- PR number, title, author, age (days open), review status
- Reviewers: requested, approved, changes-requested, no-response
- Linked issue key (from branch name or PR description)
- CI status (passing/failing/pending)

## Step 4: Categorize and rank

Group by status:
- NEEDS REVIEW: open > 1 day, no reviewer response yet
- CHANGES REQUESTED: author needs to address feedback
- APPROVED: waiting to merge (check for merge conflicts or CI failures)
- STALE: open > 5 days with no activity

## Step 5: Output

Plain text only — no markdown formatting, no ** or __ or arrows.
Use • for bullets. Section headers in ALL CAPS.

Format:
PR QUEUE — <date>

NEEDS REVIEW
• PR NNN (@author, Nd): <short title> — linked KEY

CHANGES REQUESTED  
• PR NNN (@author, Nd): <short title> — @reviewer requested changes

APPROVED / READY TO MERGE
• PR NNN (@author, Nd): <short title>

STALE (>5d no activity)
• PR NNN (@author, Nd): <short title>

BOTTLENECKS
• @reviewer has N PRs awaiting their review (list PR numbers)

Post result to Slack thread.

Exit JSON (last line):
{"status":"DONE","summary":"<N> open PRs for <M> team members: <X> need review, <Y> stale"}
