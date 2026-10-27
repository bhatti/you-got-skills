# Readiness report layout

`scripts.tickets.readiness` renders this (`reports/report.md`, `report.html`, `result.json`); use it to read the report and to build one by hand when the tool is missing.

1. **Header:** tracker, scope, ticket count, `dry-run`, and a TRUNCATED note when the scope held more tickets than the fetch limit. Then a scoreboard table (status, count, tickets) in priority order and one line: ready, needs a person, an agent can enrich.
2. **Sprint load** (sprint or milestone scope): a lane table (pending / in progress / review / done: tickets, points, share of the total, tickets without points), the open load against `--capacity` when given, **Carry-over** (tickets, points, share of the sprint, by lane, chronic, with the question for each), **Hidden rollover** (unfinished work in the active sprint that is absent from a future target), and **To decide** (one line per open flag). Points are the tool's numbers; sub-tasks are not counted.
3. **Needs attention**, grouped by status in priority order. One card per ticket, blank line between cards:
   - Header: id and title; type, status, `in-flight`, tracker status, owner, `carried over N×` (sprint names).
   - Checklist row: why, points, scope, ACs, DoD, variants as pass / partial / missing.
   - **What the ticket says:** one short verbatim quote (proves it was read).
   - **Missing:** one bullet per gap with the `searched:` evidence, plus concerns (for example unreadable points) and notes (for example "raised from missing to partial").
   - **Still needed?** when the work landed, has a duplicate or the spec drifted.
   - **Clarify:** 1-4 numbered, ticket-specific questions (Partial ACs), each with the recommended answer and why.
   - **Decisions made for you — confirm:** the recommended answers used because nobody answered.
   - **Paste-ready AC starter:** one Given/When/Then with a `Test:` line.
4. **Ready** (with points and any remaining gaps), **Sub-tasks**, **Patterns** (gap, count, tickets) and **Action items** by owner: engineers (questions, points), team (carry-over decisions: split, re-scope or drop), PM or tech lead (variant scope, blockers, unfinished work missing from the sprint), anyone with access (unreadable tickets), whoever must re-run grading (ungraded tickets), and the tickets an agent can enrich.

Never print "ACs: none" for a ticket whose text contains behavioral intent; that is Partial.

## Fetching and writing by hand

- Jira: `GET /rest/api/3/issue/<key>`; `description` is Atlassian Document Format. The issue payload embeds only a first page of comments: page `/rest/api/3/issue/<key>/comment?startAt=` until `total` is reached. Story points are a custom field whose id differs per site: look it up with `GET /rest/api/3/field` (name starts with "Story point"), and treat `null` as unset only after you read the field. Write with `PUT /rest/api/3/issue/<key>` sending the whole ADF document with the new section appended as one `panel` node whose first content node is the heading.
- GitHub: `gh issue view <n> --json number,title,body,labels,milestone,state,assignees`; edit with `gh issue edit <n> --body-file <file>` keeping the old body and the marker block.
