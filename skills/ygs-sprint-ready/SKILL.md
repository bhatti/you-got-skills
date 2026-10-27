---
name: ygs-sprint-ready
description: Use when preparing sprint planning and you need to know which candidate tickets, GitHub issues or Jira issues, are ready to pull in.
argument-hint: "[--sprint <name> | --label <label> | --milestone <name> | --jql <query> | <ticket-url|key>...] [--project P] [--repo owner/name] [--tracker github|jira] [--capacity N] [--include-done] [--limit N] [--dry-run] [--apply] [--format json]"
---

# Sprint Ready

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.
Read `~/.claude/skills/you-got-skills/skills/shared/ticket-readiness.md`. It defines the criteria, grades, statuses and write rules used below.
Read `~/.claude/skills/you-got-skills/skills/shared/tracker.md` and pass its credential gate before any tracker call.
Read `~/.claude/skills/you-got-skills/skills/shared/evidence-rules.md`.

A ticket that reaches planning without acceptance criteria or a Why costs grooming time during the sprint instead of before it. Find the gaps now and teach the author how to close them. Then show the load: story points per lane (pending / in progress / review / done), what carried over from earlier sprints, and what is still unfinished in the sprint that is about to end.

The tooling is in ai-dev-tools (`scripts/tickets`, run with `python3 -m` from its checkout, or with `PYTHONPATH` pointing at it; in a Formicary job it is already set). If the import fails, follow `ticket-readiness.md` by hand with the tracker REST API or `gh`; the grades, statuses and rules do not change.

## When NOT to use

- The sprint is already running and you need to fix one ticket: use `/ygs-triage` or `/ygs-enrich-ticket <ticket>`.
- No tracker is configured: BLOCKED with the credential gate message.

## Modes

| Invocation | Behavior |
|------------|----------|
| `--dry-run` | Grade and report only. No enrichment, no writes (wins if `--apply` is given too) |
| Interactive session | Report, then offer enrichment and confirm before each batch |
| `--apply` | Enrich every auto-enrichable ticket without asking |
| Headless (`YGS_HEADLESS=1`, or nobody can answer) without `--apply` | Same as `--dry-run`; say so in the report |

## 1. Fetch

First confirm the credentials, so a sprint-sized run does not fail after the research is done:
```bash
python3 -m scripts.tickets.fetch --check
```
If it fails, show its message and stop. Do not look for credentials anywhere else.

```bash
mkdir -p ready-out reports
python3 -m scripts.tickets.fetch --sprint "<name>" --out ready-out/tickets.json      # or --label / --milestone / --jql / <refs>
```
Pass `--repo owner/name` and `--tracker github|jira` to every tool call when they were given (in a Formicary job the environment already carries them). Besides `tickets.json` it writes one plain-text file per ticket, `ready-out/tickets/<id>.txt` (`<id>` is the ticket id with `#` written as `gh-`).
A sprint or milestone name may contain spaces and Slack strips the quotes: take everything after `--sprint` (or `--milestone`, `--label`) up to the next `--flag` as the name and pass it quoted. GitHub has no sprints: `--milestone` is the sprint (or `--label`), and `#42` style refs need `--repo owner/name` or `GH_ORG` and `GH_REPO`. Jira: `--sprint` with `--project` (default `JIRA_PROJECT`). The tracker follows the usual rule: `--tracker` flag, then the URL, then `DEFAULT_TRACKER`. A sprint or milestone scope always reads the finished tickets too, because the point totals per lane and the carry-over need them; they are only graded with `--include-done`. `count` includes them, `evaluated` is what gets graded. `load` in the fetch output holds the lane totals, carry-over and rollover: use those numbers as given.

- Exit 2 or 3, or `count` 0: stop. Report the exact message; BLOCKED for access or credentials. Never fall back to treating unreadable fields as unset.
- `truncated: true` means the scope held more tickets than `--limit`: re-run with a higher `--limit` before trusting the counts.
- A ticket with `fetch_error` or `access_denied` is `UNREADABLE`: list it, do not grade it.
- Show the list (key, title, status, owner) and the count. Interactively, confirm the scope first.

## 2. Grade

Read each ticket's plain-text file `ready-out/tickets/<id>.txt` (the complete description, environment and every comment, and a sub-task's parent), in chunks if it is large, until its last line: never grade from `signals` or a clipped view (`ticket-readiness.md`, "Read the whole ticket"). The `signals` are hints, not verdicts. Apply the criteria and the evidence rule and write `ready-out/grades.json`. More than about 8 tickets: dispatch workers of 5 tickets each (`~/.claude/skills/you-got-skills/skills/shared/subagent-dispatch.md`). Give each worker, self-contained: `ticket-readiness.md` (criteria, evidence rule and "Read the whole ticket"), the paths of its tickets' `ready-out/tickets/<id>.txt` files, and the instruction to read each completely and to return its part of `grades.json` in the schema of that file. If you cannot dispatch workers, grade the tickets yourself, one at a time, with the same rules. Merge the parts; a ticket still missing afterwards is `UNGRADED`, so re-dispatch it once. Do not grade story points, blocked, sub-task, in-flight or plan presence yourself: they come from the fetch data.

Carry-over tickets that are pending or in progress: check whether the work already landed before asking anyone to spend another sprint on it (`ticket-readiness.md`, "Is the work still needed?") and record it in `still_needed`.

## 3. Verdict and report

```bash
python3 -m scripts.tickets.readiness --tickets ready-out/tickets.json --grades ready-out/grades.json [--capacity N] [--dry-run] --out-dir reports
```
`--capacity N` is the team's capacity in story points, compared with the open (not done) load.
It reads `.ygs/readiness.yml` from the working directory or the checked-out repository (`$CODEBASE_DIR`) when present (`--config` only for another path; without the file the variant check is skipped). Exit 2 means an input is invalid (the message names the bad grade, duplicate id or file): fix `grades.json` and run it again; do not edit the report by hand. It writes `reports/report.md`, `report.html`, `result.json` and `slack.md` (the short summary the job posts to Slack; leave it as written) and prints the status line. Show `report.md` (layout: `~/.claude/skills/you-got-skills/skills/ygs-sprint-ready/references/report.md`). Do not edit statuses by hand.

## 4. Enrich (skip with `--dry-run` or when headless without `--apply`)

The tickets to enrich are the rows of `reports/result.json` with `auto_enrichable: true`; the tool already excluded finished, In Review and other no-enrich tickets, so do not re-derive it from status names. `BLOCKED`, `NEEDS_STORY_POINTS`, `HAS_VARIANT_HOLES`, `UNREADABLE` and `UNGRADED` are action items for a person.

1. Interactive: show the gaps and ask once ("enrich all N" or per ticket). `--apply`: no question.
2. For each ticket, `Read ~/.claude/skills/you-got-skills/skills/ygs-enrich-ticket/SKILL.md` and follow its steps 2 and 3 using the fetch and grades you already have (do not fetch, grade or render a report again, and do not run its step 4: you own `ready-out/actions.json` and the final line). Previews go to `reports/<id>/`, where `<id>` is the ticket id with `#` written as `gh-`.
3. Append every JSON result line `scripts.tickets.write` prints to the list in `ready-out/actions.json`. Exit 3 from `write` (`blocked_points`, `ineligible`) is an expected outcome: record it and go on.

## 5. Re-evaluate and finish

Re-run step 3 with `--actions ready-out/actions.json` (skip when nothing was enriched) so enriched tickets show `ENRICHED`, `ENRICHED_PENDING_POINTS` or `IMPL_PLAN_LOW_CONFIDENCE`. Finish with the action items grouped by owner (engineers: questions and points; team: carry-over decisions; PM or tech lead: variant scope, blockers and unfinished work missing from the sprint) and suggest `/ygs-sprint-plan` once READY covers the target capacity.

The last line of output is the status line printed by `scripts.tickets.readiness`. `--format json`: print `reports/result.json` before it. If you must stop before the tool can run, print `{"status":"ERROR","signal":"BLOCKED","summary":"<exact reason>"}`.
