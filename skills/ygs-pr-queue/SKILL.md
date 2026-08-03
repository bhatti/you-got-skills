---
name: ygs-pr-queue
description: Show open PRs authored by your primary sprint board team members — same
  board as standup. Plain-text output. Scoped to sprint team authors only.
---

# PR Queue — Sprint Board View

**Principle:** Only show PRs authored by sprint team members on your primary board. Do NOT show PRs where you are listed as a reviewer by an auto-assignment bot.

## Step 1: Init

Follow `~/.claude/skills/you-got-skills/skills/shared/init.md`.

## Step 2: Identify primary sprint board and team

Same logic as ygs-standup:
- Load active sprint(s) from the tracker (Jira/GitHub)
- Primary board = the board whose sprint includes issues assigned to you (the current user)
- Team members = unique assignees with issues in THAT board's sprint
- If STANDUP_TEAM_MEMBERS is set, use that list directly

## Step 3: Fetch open PRs — authored by sprint team ONLY

Query open PRs where **author** is one of the sprint team members identified in Step 2.

**STRICT RULE: Do NOT include PRs where a team member is merely listed as a reviewer.**
Auto-assignment tools (goatbot, code-owners) add people as reviewers to hundreds of PRs — those are irrelevant noise. Only PRs where a sprint team member is the **author** are included.

For each PR collect:
- PR number, title, author (first name only), age in days
- Reviewers who have approved
- Reviewers who have requested changes
- Reviewers assigned but no response yet
- Linked Jira issue key from branch name or PR title

## Step 4: Categorize by review status

Group into:
- NEEDS REVIEW: open >1d, no reviewer has responded yet
- CHANGES REQUESTED: reviewer left feedback, author needs to act
- APPROVED: all reviewers approved, ready to merge
- STALE: open >5d with no reviewer activity at all

Skip any empty category.

## Step 5: Output — STRICT FORMAT RULES

**FORBIDDEN (violation = wrong answer):**
- No markdown tables (no | pipe characters, no --- dividers)
- No ** bold, no __ underline, no ` backticks, no # headings
- No → arrows, no > blockquotes, no - list bullets
- No :emoji: in the PR lines (use 🔴 🟡 Unicode emoji only for RISKS section)

**REQUIRED:**
- Bullets: • character only
- Section headers: ALL CAPS
- One PR per line, max 120 chars
- Format each line: • PR NNN (@FirstName, Nd): short title [KEY if known]

**Exact template — copy structure, fill data:**

PR QUEUE — <date>
BOARD: <BoardName> Sprint <N>
TEAM: <FirstName FirstName FirstName ...>

NEEDS REVIEW
• PR NNN (@FirstName, Nd): short title [KEY]
• PR NNN (@FirstName, Nd): short title [KEY] — no reviewer assigned

CHANGES REQUESTED
• PR NNN (@FirstName, Nd): short title [KEY] — @reviewer asked for changes

STALE (>5d no activity)
• PR NNN (@FirstName, Nd): short title [KEY]

BOTTLENECKS
• @FirstName: N PRs waiting on review (NNN NNN NNN)

Post result to Slack thread (use SLACK_THREAD_TS if set, else post to channel).

Exit JSON (last line, required):
{"status":"DONE","summary":"<N> open PRs by <M> sprint team members: <X> need review, <Y> stale"}
