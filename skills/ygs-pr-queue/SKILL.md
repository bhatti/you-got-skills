---
name: ygs-pr-queue
description: Format sprint board PR queue from pre-gathered pr_queue.json. Plain-text output matching standup format.
---

# PR Queue — Sprint Board View

The data is already provided below in the prompt under "Pre-Fetched PR Data".
DO NOT make any API calls. DO NOT use any tools. Format ONLY the provided data.

Read `~/.claude/skills/you-got-skills/skills/shared/output-format.md` for shared row templates.

## Output Format — STRICTLY ENFORCED

RULES (violation = wrong answer):
1. NO markdown: no **, no __, no `, no #, no ->, no →
2. Bullets: • only (never -)
3. Section headers ALL CAPS on their own line
4. One PR per bullet line
5. Every PR appears EXACTLY ONCE — no duplicate sections
6. Include BOTH Jira link AND PR link on each line using Slack mrkdwn: `<url|PROJ-NNN>` and `<url|PR #NNN>`
7. If jira_url is empty, omit the Jira link; if pr_url is empty, omit the PR link

EXACT TEMPLATE (Slack mrkdwn):

PR QUEUE — <sprint_name> — <date>

• <jira_url|PROJ-NNN> <pr_url|PR #NNN> @FirstName (Nd) short description ≤40 chars approved-by: @Name, @Name | pending: @Name, @Name

If posting to stdout only: replace `<url|KEY>` with `KEY (url)`.

Organise PRs into sections (skip empty sections, no duplicate entries):

APPROVED — READY TO MERGE
(PRs that have at least one approval)

NEEDS REVIEW (open >1d, no approvals yet)
(PRs open more than 1 day with zero approvals)

STALE / AT RISK (open >5d, no approvals)
(PRs open more than 5 days with zero approvals — flag as risk)

IN REVIEW (everything else with pending reviewers)
(PRs with pending reviewers but not yet approved or stale)

Guidelines:
- Use first name only (e.g. "Shahzad Bhatti" → "Shahzad")
- If jira_summary is more descriptive than the PR title, use it for description
- Show approved_by as "approved-by: @Name, @Name" (comma-separated, first names only)
- Show pending reviewers as "pending: @Name, @Name" (first names only, max 5 then "...")
- If no reviewers at all: show "no reviewers"
- STALE PRs should also appear under STALE even if they have some approvals (age > 5d is a risk)

Exit JSON (last line, required):
{"status":"DONE","summary":"<N> sprint PRs: <X> approved, <Y> needs review, <Z> stale"}
