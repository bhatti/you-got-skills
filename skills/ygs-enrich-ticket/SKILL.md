---
name: ygs-enrich-ticket
description: Use when a GitHub or Jira ticket lacks testable acceptance criteria or an implementation plan, before sprint planning or before assigning the work.
argument-hint: "[<ticket-url|key> | --sprint <name> | --label <label> | --milestone <name>] [--project P] [--repo owner/name] [--tracker github|jira] [--acs-only | --plan-only] [--dry-run] [--apply] [--force] [--limit N] [--format json]"
---

# Enrich Ticket

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.
Read `~/.claude/skills/you-got-skills/skills/shared/ticket-readiness.md`. It defines the grades, statuses and the write rules used below.
Read `~/.claude/skills/you-got-skills/skills/shared/ac-format.md`.
Read `~/.claude/skills/you-got-skills/skills/shared/tracker.md` and pass its credential gate before any tracker call.
Read `~/.claude/skills/you-got-skills/skills/shared/evidence-rules.md`.

A ticket enriched before planning costs half an hour; the same ticket enriched mid-sprint, with an agent guessing at scope, costs a sprint.

The tooling is in ai-dev-tools (`scripts/tickets`, run with `python3 -m` from its checkout, or with `PYTHONPATH` pointing at it; in a Formicary job it is already set). If the import fails, follow `ticket-readiness.md` by hand; never write plain text over a Jira description, post a comment instead.

## When NOT to use

- The ticket is blocked, a sub-task, finished or in a no-enrich status (In Review): report its gaps, change nothing. `scripts.tickets.write` refuses these anyway.
- The code base is not available: the plan would be noise. Write ACs only (`--acs-only`) and say why.

## Write policy

Nothing is written without `--apply`. Interactive: show the preview and confirm. Headless (`YGS_HEADLESS=1`, or nobody can answer) without `--apply`: preview only. `--dry-run` always previews. A preview is the full text, saved as `reports/<id>/acs.md` and `reports/<id>/plan.md` (`<id>` is the ticket id with `#` written as `gh-`, so `PROJ-123` or `gh-42`), so a dry run shows exactly what would be written. Story points are never written; suggest, do not set.

## 1. Fetch and grade

Called from `ygs-sprint-ready`, you already have the fetch and the grades: use them and skip to step 2. Alone (one ticket or a batch), confirm the credentials first so a batch does not fail after the research is done (`python3 -m scripts.tickets.fetch --check`; if it fails, show its message and stop, and do not look for credentials anywhere else), then:

```bash
mkdir -p ready-out/batch reports
python3 -m scripts.tickets.fetch <ticket|--sprint|--label|--milestone ...> --out ready-out/batch/tickets.json
```
Pass `--repo owner/name` and `--tracker github|jira` to every tool call when they were given (in a Formicary job the environment already carries them). Each ticket also gets a plain-text file, `ready-out/batch/tickets/<id>.txt`.
Read each ticket's plain-text file (the complete description, environment and every comment; in chunks if needed, to its last line; never clipped, never from `signals` alone), grade it as in `ticket-readiness.md` (evidence rule included) into `ready-out/batch/grades.json`, then run `python3 -m scripts.tickets.readiness --tickets ready-out/batch/tickets.json --grades ready-out/batch/grades.json --dry-run --out-dir ready-out/batch/gap` and show the gap report (`ready-out/batch/gap/report.md`). From here on `<id>` is one ticket's id with `#` written as `gh-` (`PROJ-123`, `gh-42`); each ticket's previews go to `reports/<id>/`. `UNREADABLE`, `BLOCKED`, no-enrich or sub-tasks: stop for that ticket. A ticket that already has the generated sections is skipped unless `--force`. With `--dry-run` continue only to produce the previews; no writes.

## 2. Acceptance criteria (only when ACs are Missing or Partial; skip with `--plan-only`)

Skip when they are already testable. When the fetch shows story points unset, generate the preview anyway and still call `write` (a refusal is the same with or without `--apply`): it prints `blocked_points` and the exact line to record. Say the points are the next step.

- **Missing:** write them from the ticket, its parent and the research below.
- **Partial:** do not discard what exists. Quote each statement, keep the author's wording and intent, and convert it into a full AC. Questions follow `ticket-readiness.md` ("Partial is the common case"): at most 4, each with your recommended answer and why, and never one the code already settles (state that as an assumption). Interactive: ask them first when the answers change an AC materially, write each answer into the AC, Out of scope or DoD it affects (never append a Q&A section), and repeat until no open point remains. Headless, dry run, or the engineer cannot answer: write your recommended answer into the ACs and list each one under a final `**Decisions made for you — confirm**` line, so the ticket shows what was decided for them.
- **Still needed?** First check the work has not landed, has no duplicate and the spec matches the code (`ticket-readiness.md`, "Is the work still needed?"). If it is done or the spec is wrong, say so in the report and write no ACs.
- Format, minimums (3 ACs, a failure case, a boundary case, compatibility when versions or stored configuration change), DoD and the out-of-scope block: `ac-format.md`. Each AC: Given names a real context, Then is observable without reading the source (not a log line), `Test:` names the test file and the command, or exact steps.
- If the ticket implements a task from a design breakdown, read that task file too; its status and checkboxes are often stale, so trust the code and note a file that disagrees.
- Ground them with the bounded research in `~/.claude/skills/you-got-skills/skills/ygs-enrich-ticket/references/research.md` (design docs, source, existing tests).
- If `.ygs/readiness.yml` defines variants, add an AC per in-scope variant and mark the others explicitly out of scope or as an open question; never invent coverage.

Write the section to `reports/<id>/acs.md` in plain markdown (headings, bullets, `**bold**`, `` `code` ``, links; no tables), then:
```bash
python3 -m scripts.tickets.write --ref <ticket> --kind acs --section-file reports/<id>/acs.md [--apply] [--force]
```
It refuses to write while story points are unset (`blocked_points`), appends once, preserves the description, reads it back, and falls back to a comment on failure. Print its JSON line as is.

## 3. Implementation plan (one worker per ticket; skip with `--acs-only`)

Write a plan only when the ACs pass or were just written; if they are blocked or still partial, skip the plan and say why. Dispatch a self-contained worker per ticket (`~/.claude/skills/you-got-skills/skills/shared/subagent-dispatch.md`; if you cannot dispatch workers, do each ticket yourself with the same steps): the path of the ticket's plain-text file (it holds the complete description and every comment), the repository root (`$CODEBASE_DIR` in a Formicary job), `~/.claude/skills/you-got-skills/skills/ygs-enrich-ticket/references/research.md` and `~/.claude/skills/you-got-skills/skills/shared/confidence-rubric.md`; no session context. Each worker traces the code from the entry point to the data layer, reads the last-touch history of the files it names, scores confidence with the rubric, and drafts:

```markdown
## Implementation Plan (generated)
**Approach:** one or two sentences
**Files to change:** `path` - what and why
**Dependencies and risks:** ...
**Suggested skills:** ...
```

| Score | Action |
|-------|--------|
| HIGH | `scripts.tickets.write --kind plan` |
| MEDIUM | Same, with the caveat blockquote from the rubric as the first lines |
| LOW | Write nothing to the description. Post one comment saying what is missing: `scripts.tickets.write --kind plan --comment-only --section-file ...`. The ticket becomes `IMPL_PLAN_LOW_CONFIDENCE`, not "needs a plan" again |

## 4. Finish

Collect the JSON result lines that `scripts.tickets.write` printed into `ready-out/actions.json` as a list (append when the file exists), unchanged. Actions: `appended`, `replaced`, `exists`, `commented`, `dry_run` (exit 0); `blocked_points`, `ineligible`, `unreadable` (exit 3, expected: record and go on); `failed`, `written_unverified` (exit 1: record, do not retry, look at the ticket). With `--format json` print that list.

Called from `ygs-sprint-ready`, stop here: it owns the final line. Alone, run the command below on **every** path, also when every ticket was skipped, unreadable, ineligible or already enriched: the job fails a run that ends without the `reports/result.json` it writes. Run `python3 -m scripts.tickets.readiness --tickets ready-out/batch/tickets.json --grades ready-out/batch/grades.json --actions ready-out/actions.json [--dry-run] --out-dir reports` (add `--dry-run` when nothing was applied, so the report says so; the job posts `reports/report.md` and `reports/slack.md`; keep them from this command) so the ticket states reflect what was written, summarise what was written, skipped and assumed (and any significant unknowns found in the code), and end with that command's status line as the last line of output. If you must stop before it can run, print `{"status":"ERROR","signal":"BLOCKED","summary":"<exact reason>"}`. The previews you saved under `reports/<id>/` appear in that report's Previews section. Suggest `/ygs-sprint-ready` to validate, then `/ygs-implement`.
