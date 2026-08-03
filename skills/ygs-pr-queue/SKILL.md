---
name: ygs-pr-queue
description: Format sprint board PR queue from pre-gathered pr_queue.json data. Plain-text output, no markdown.
---

# PR Queue — Sprint Board View

Sprint team PRs have already been gathered and written to `/workspace/pr_queue.json`.
Do NOT make any API calls. Read and format ONLY the data in that file.

## Step 1: Read the data

```bash
cat /workspace/pr_queue.json
```

The file contains:
- `sprint`: sprint name
- `team`: list of team member display names
- `prs`: list of PRs, each with: id, title, author, url, age_days, jira_key, reviewers (pending), approved_by, changes_requested_by

## Step 2: Categorize each PR

- APPROVED: approved_by is non-empty AND reviewers (pending) is empty
- CHANGES REQUESTED: changes_requested_by is non-empty
- NEEDS REVIEW: reviewers (pending) is non-empty, no approvals yet, age_days > 1
- STALE: age_days > 5 AND approved_by is empty

A PR can appear in multiple categories (e.g. STALE + NEEDS REVIEW). Show in the most urgent category only.

## Step 3: Output — STRICT FORMAT RULES

FORBIDDEN (violation = wrong answer):
- No markdown tables (no | pipe characters)
- No ** bold, no __ underline, no backticks, no # headings
- No -> arrows, no > blockquotes
- No - dash bullets — use • only

REQUIRED:
- Section headers: ALL CAPS
- One PR per line: • PR NNN (@FirstName, Nd): short title [JIRA-KEY] — reviewer status
- URL on same line if it fits, else skip
- Show first name only from display_name (e.g. "Shahzad Bhatti" → "Shahzad")

Exact template:

PR QUEUE — <sprint_name> — <date>
TEAM: <FirstName FirstName FirstName ...>

NEEDS REVIEW (>1d, no approval)
• PR NNN (@FirstName, Nd): short title [KEY] — no reviewers assigned
• PR NNN (@FirstName, Nd): short title [KEY] — @ReviewerA @ReviewerB pending

CHANGES REQUESTED
• PR NNN (@FirstName, Nd): short title [KEY] — @Reviewer asked for changes

APPROVED — WAITING TO MERGE
• PR NNN (@FirstName, Nd): short title [KEY] — approved by @ReviewerA

STALE (>5d, no approval)
• PR NNN (@FirstName, Nd): short title [KEY] — @Reviewers pending

BOTTLENECKS
• @FirstName: N PRs waiting on review (NNN NNN)

Skip any empty category.

Post result to Slack thread (use SLACK_THREAD_TS if set, else post to channel).

Exit JSON (last line, required):
{"status":"DONE","summary":"<N> PRs for <sprint>: <X> need review, <Y> stale, <Z> approved"}
