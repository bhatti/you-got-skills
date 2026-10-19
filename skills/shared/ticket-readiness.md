# Ticket Readiness

Shared by: `ygs-sprint-ready`, `ygs-enrich-ticket`. The one definition of how a ticket is graded, which status it gets, and how generated text is written back. Works for GitHub issues and Jira issues alike.

## Who decides what

| Decision | Made by | Why |
|----------|---------|-----|
| Story points, blocked, sub-task, in-flight, fetch errors, whether a plan exists | `scripts.tickets.fetch` (data, never guessed) | A model cannot see a custom field or an unreadable ticket |
| Why, scope, ACs, DoD, variant coverage | You, as Pass / Partial / Missing in `grades.json`, with quoted evidence | Needs reading comprehension |
| Status, counts, report, JSON, final status line | `scripts.tickets.readiness` | Same grades always give the same verdict; do not rewrite it |
| Writing text to a ticket | `scripts.tickets.write` | Idempotent, preserves existing content, verifies by re-reading |

The tooling is in ai-dev-tools (`scripts/tickets`, run with `python3 -m`). Without it, do the same by hand: read the ticket through the tracker REST API or `gh` (see `tracker.md`), keep the same fields and statuses, and never write plain text over a Jira description (it is Atlassian Document Format; a plain write destroys tables, links and formatting). If you cannot write safely, post the text as a comment instead.

## Criteria and grades

| Criterion | Pass | Partial | Missing |
|-----------|------|---------|---------|
| Why | States the user or business problem | Problem implied by the title or a link only | No context (parent has none either) |
| Scope | Explicit in-scope **and** out-of-scope | One side only, or implementation notes | None |
| ACs | 3+ Given/When/Then, each with an observable outcome **and** a `Test:` line | Some behavior is stated (AC section, bullets, repro steps, requirements) but not Given/When/Then or lacks `Test:` | No behavioral statement at all |
| DoD | Runnable test command or specific manual steps | "Needs tests" in general | None |
| Variants | Every in-scope variant has an AC and a verification | Some covered | Silent (only when the project defines variants) |

Story points are not graded by you: the fetch result says `value`, `source` and `known`. `known: false` means the field could not be read; say so, never "unset". A value outside the scale (default 1, 2, 3, 5, 8, 13) or 21 and above (split it) is flagged.

### Read the whole ticket

Grade from the **complete** description and **every** comment, never from `signals`, a summary or a clipped view: a clipped acceptance criterion is a wrong verdict. The fetch tool never truncates anything (`description_chars` is the length it holds; every comment is kept whole, including pages the tracker returns separately). A ticket whose comments could not be read completely is `UNREADABLE`, never graded on part of its text. Read each ticket from its plain-text file (`ready-out/tickets/<id>.txt`, named in `text_file`; real line breaks, nothing cut): `tickets.json` holds the same text as single very long lines that a file reader may cut off. A file too large for one read is read in consecutive chunks (offset and limit) until its last line. A sub-task's file includes its parent's description. Also read `environment` when it is not empty. Quote verbatim in `says`, `evidence`, an AC or a plan (never paraphrase or clip a quote), but read all of the ticket. If you could not read something in full, say so in the grade's `evidence` instead of guessing.

### Evidence rule (no false positives)

Before grading anything Missing or Partial, search the **whole** ticket: description, every heading, every comment, the environment field, the parent for Why and scope. Specs hide under other headings: `Outcome`, `Requirements`, `Expected result`, `Verification`, `Tests`, `Done when`, `Out of scope`, `Steps to reproduce`. Content there counts (`Verification` is a DoD, `Out of scope` is scope, `Requirements` is behavior). For every gap write in `evidence` what you searched and a short quote ("searched: Outcome, Verification; no observable HTTP or UI result stated"). When unsure grade **Partial and ask a question**, not Missing. `scripts.tickets.readiness` raises a Missing grade to Partial when the fetch signals show behavior text, so a careless Missing cannot send a ticket to "write from scratch".

### Partial is the common case: clarify, do not rewrite

Quote what exists, name what is undefined as 1-4 questions answerable in one line (trigger, observable result, empty or error behavior, scope of the change), and give one paste-ready `ac_starter` in the author's own words. Questions come from this ticket, never generic ("add ACs").

## `grades.json`

```json
{"tickets": [{
  "id": "PROJ-123",
  "checks": {"why": "pass", "scope": "partial", "acs": "partial", "dod": "missing", "variants": "na"},
  "says": "short quote proving you read it",
  "evidence": {"acs": "searched: Requirements, Verification; bullets describe the how, no observable result"},
  "questions": ["What does the filter match: name only, or also id?"],
  "ac_starter": "Given ..., when ..., then ...; Test: ...",
  "variant_holes": [],
  "suggested_points": null
}]}
```
`variants` is `na` unless the project defines variants (`.ygs/readiness.yml`, key `variants`). `suggested_points` is advice for a human; nothing writes points. Sub-tasks need only `why`.

## Statuses (highest priority wins)

| # | Status | Condition |
|---|--------|-----------|
| 1 | `UNREADABLE` | Fetch failed or access denied. Never graded, never guessed |
| 2 | `UNGRADED` | The ticket was fetched but no grade was recorded for it. Never ready by default; grade it |
| 3 | `BLOCKED` | Blocked label or unresolved blocking link, or no usable context |
| 4 | `NEEDS_AC_WORK` | ACs Missing |
| 5 | `NEEDS_AC_CLARIFICATION` | ACs Partial |
| 6 | `NEEDS_STORY_POINTS` | Points unset (known) or off-scale |
| 7 | `NEEDS_IMPL_PLAN` | ACs Pass, points set, no plan |
| 8 | `HAS_VARIANT_HOLES` | A plausible variant has nothing written (explicit out-of-scope never counts) |
| 9 | `READY` | Everything passes; Partial or Missing Why, scope or DoD stay listed as `gaps` |

Sub-tasks: `SUBTASK_OK` or `SUBTASK_NEEDS_CONTEXT` (no description, no clear title action, and the parent gives no Why). After enrichment: `ENRICHED`, `ENRICHED_PENDING_POINTS` (ACs generated, write blocked until points are set; a dry run that would be blocked reports the same), `IMPL_PLAN_LOW_CONFIDENCE` (a comment was posted, so the ticket does not loop). A ticket also lacking points keeps its AC status plus a note.

In-flight tickets (In Progress or In Review) are graded and tagged. Tickets finished, or in one of `no_enrich_statuses` (default In Review, Done), are reported but never rewritten, and `scripts.tickets.write` refuses them (as it refuses blocked tickets and sub-tasks). GitHub has no workflow states: a closed issue counts as finished, and a label equal to a configured status (for example `in review`) counts as that status. Done tickets are excluded from a fetch unless `--include-done`.

`auto_enrichable` = `NEEDS_AC_WORK`, `NEEDS_AC_CLARIFICATION`, `NEEDS_IMPL_PLAN` and not no-enrich. `needs_human` = `UNREADABLE`, `UNGRADED`, `BLOCKED`, `NEEDS_STORY_POINTS`, `HAS_VARIANT_HOLES`, `SUBTASK_NEEDS_CONTEXT`, `IMPL_PLAN_LOW_CONFIDENCE`, `ENRICHED_PENDING_POINTS`.

## Project config (optional): `.ygs/readiness.yml`

```yaml
points_scale: [1, 2, 3, 5, 8, 13]
variants:                         # omit to skip the variant check entirely
  - {name: "single node", description: "one-process deployment"}
no_enrich_statuses: ["In Review", "Done"]
```

## Writing to tickets

| | Jira | GitHub |
|---|------|--------|
| Acceptance criteria | One panel holding the heading `Acceptance Criteria (generated)` and the text, appended to the description | Same text between `<!-- ygs:acceptance-criteria:start -->` and `:end -->` in the body |
| Implementation plan | Panel with the heading `Implementation Plan (generated)` | Markers `ygs:impl-plan` |
| Story points | Never written | Never written |

- **Dry run is the default.** Anything is written only with `--apply`; an interactive session also confirms per batch. You are **headless** when `YGS_HEADLESS=1` is set (run `echo $YGS_HEADLESS` once at the start) or you cannot ask a question and get an answer in this session (a Formicary job): never wait for a confirmation then; without `--apply` preview and report what would be written. When in doubt, preview.
- Existing content is preserved: the section is one panel (Jira) or one marked block (GitHub), so `--force` replaces exactly that and never text a person added around it; text a person wrote under the same heading is never replaced. A second run finds the section and does nothing (`exists`); an implementation plan written by hand counts too.
- Write the section in plain markdown: headings, bullets (nested by indentation), numbered lists, `**bold**`, `` `code` ``, links, fenced code. Tables become code blocks; avoid them.
- ACs are not written while story points are unset or unreadable (the ACs are still previewed). Write a plan only when the ACs pass or were just written. A plan is written per the confidence rubric.
- Every write is read back and compared (the rest of the ticket unchanged, every word of the section present). A failed write leaves the ticket untouched and the text is posted as a comment.
- `scripts.tickets.write` prints one JSON result line. Exit 0: `appended`, `replaced`, `exists`, `commented`, `dry_run`. Exit 3 is an expected outcome, not an error to retry: `blocked_points`, `ineligible`, `unreadable`; record the line and go on. Exit 1 `failed` or `written_unverified` (accepted by the tracker but not reading back as expected: look at the ticket): record it, report DONE_WITH_CONCERNS, do not retry. A repeated `--comment-only` answers `commented` with "already posted". The result says `redacted: true` when secrets were removed from the text.
- "Plan present" is true for either form on either tracker (generated heading or markers).

## Report and JSON

Verdict first: a scoreboard (status, count, tickets), then one card per ticket that needs work (type, status, in-flight, points, owner; checklist row; what the ticket says; what is missing; questions; AC starter), then READY, sub-tasks, cross-ticket patterns and action items by owner. No wide gap tables. `readiness` writes `reports/report.md`, `report.html`, `result.json` (`verdict` text, `summary` counts, one row per ticket) and `slack.md` (the short mrkdwn summary the job posts to Slack), adds a Previews section for any `reports/<id>/acs.md` / `plan.md` the enrich skill saved, and prints the status line:

`{"status":"DONE","signal":"DONE_WITH_CONCERNS","total":14,"ready":9,"needs_human":3,"auto_enrichable":2,"summary":"..."}`

`DONE` when every ticket is `READY`, `ENRICHED` or `SUBTASK_OK`; `DONE_WITH_CONCERNS` when any ticket is in another status (the summary names each status and count); `BLOCKED` (`status` ERROR) when no ticket could be read. Definitions: `completion-signals.md`; line format: `evidence-rules.md`.
