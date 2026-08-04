---
name: ygs-pr-queue
description: Format sprint board PR queue from pre-gathered pr_queue.json. Plain-text output matching standup format.
---

# PR Queue — Sprint Board View

The data is already provided below in the prompt under "Pre-Fetched PR Data".
DO NOT make any API calls. DO NOT use any tools. Format ONLY the provided data.

## Output Format — STRICTLY ENFORCED

RULES (violation = wrong answer):
1. NO markdown: no **, no __, no `, no #, no >, no ->, no →
2. Bullets: • only (never -)
3. Section headers ALL CAPS on their own line
4. One PR per bullet line

EXACT TEMPLATE:

PR QUEUE — <sprint_name> — <date>

• <JIRA-KEY>  PR <NNN>  @<FirstName> (<N>d)  <short description ≤40 chars>  <reviewer status>
  (reviewer status = "no reviewers" | "approved by @Name" | "@Name @Name pending" | "changes requested by @Name")

Organise PRs into sections (skip empty sections):

NEEDS REVIEW (open >1d, no approvals)
• <JIRA-KEY>  PR <NNN>  @<FirstName> (<N>d)  <description>  @Reviewer @Reviewer pending

APPROVED — READY TO MERGE
• <JIRA-KEY>  PR <NNN>  @<FirstName> (<N>d)  <description>  approved by @Name

STALE (open >5d, no approvals)
• <JIRA-KEY>  PR <NNN>  @<FirstName> (<N>d)  <description>  no reviewers

ALL OPEN PRS
• <JIRA-KEY>  PR <NNN>  @<FirstName> (<N>d)  <description>  <reviewer status>

Guidelines:
- Use first name only (e.g. "Shahzad Bhatti" → "Shahzad")
- If jira_summary is more descriptive than the PR title, use it for description
- Every open PR must appear in ALL OPEN PRS even if already in another section
- PR link: show PR number plainly (no URL, no markdown)

Exit JSON (last line, required):
{"status":"DONE","summary":"<N> sprint PRs: <X> need review, <Y> approved, <Z> stale"}
