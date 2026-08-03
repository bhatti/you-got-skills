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
4. One PR per bullet, include: PR number, author first name, age, Jira key, reviewer status

EXACT TEMPLATE:

PR QUEUE — <sprint_name> — <date>

OPEN PRS
• PR <NNN> (@<FirstName>, <N>d): <short title> [<JIRA-KEY>] — <reviewer status>
  (reviewer status = "no reviewers" / "<N> approved" / "<N> pending: @Name @Name" / "changes requested by @Name")

NEEDS REVIEW (open >1d, no approvals yet)
• PR <NNN> (@<FirstName>, <N>d): <short title> [<JIRA-KEY>] — @Reviewer @Reviewer pending

APPROVED — READY TO MERGE
• PR <NNN> (@<FirstName>, <N>d): <short title> [<JIRA-KEY>] — approved by @Name

STALE (open >5d, no approvals)
• PR <NNN> (@<FirstName>, <N>d): <short title> [<JIRA-KEY>] — no reviewers assigned

Skip any section that has no PRs.
Use first name only from display_name (e.g. "Shahzad Bhatti" → "Shahzad").

Exit JSON (last line, required):
{"status":"DONE","summary":"<N> sprint PRs: <X> need review, <Y> approved, <Z> stale"}
