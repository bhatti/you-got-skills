# Evidence Rules — what counts as proof in QA and flaky-test work

**Iron law: no verdict without fresh evidence from this run.** Output remembered from an earlier run, a PR author's claim ("all tests pass") or a green badge is a lead, not evidence. Run the command, read the output and exit code, then state the result.

## Outcome vocabulary

Use exactly these per case. Never invent "mostly passed".

| Outcome | Meaning |
|---------|---------|
| `pass` | Ran, and observed output matched the expectation |
| `fail` | Ran, and observed output contradicted the expectation (product problem) |
| `blocked` | Could not run it (environment down, missing credential, unreachable baseline) |
| `not-run` | Deliberately skipped (state why) |
| `inconclusive` | Ran but evidence is ambiguous (flaky, partial output) |

## Non-negotiable rules

1. **Zero executed cases is never pass or clean.** Report BLOCKED and say what prevented execution.
2. **Blocked and skipped never count as passed.** Health scores exclude them and are labelled "partial coverage".
3. **Infrastructure is not product.** Connection refused, HTTP 000, DNS failure, image pull failure, missing dependency and daemon-not-running are `blocked`, reported separately from `fail`. They are never a regression.
   The same applies to the run's own prerequisites: prefer REST, `gh`, `curl` and the project's own commands; a vendor CLI whose profile may exist only on some machines (`acli`, `jira`) is an optional fallback used when it demonstrably works, never an assumption. If a tool, credential or daemon this run needs is missing or unconfigured, stop with BLOCKED naming it and how to supply it; never continue with partial data or report a clean result.
4. **Every finding cites evidence:** `file:line`, or the exact command plus the observed output (trimmed). A finding with no citation is dropped.
5. **Do not trust reports.** Check an author's or tool's claim against the diff and a fresh run.
6. **Declare what was not judged.** End with a "Not judged" list (areas, platforms, criteria you could not verify) so omissions are visible.
7. **Never loosen a test to make it pass.** Do not widen timeouts, add retries or delete assertions as a "fix" without evidence of the root cause.
8. **Absence of a failure is weak.** "0 failures in N runs" is stated as `0/N` with the upper bound it implies, at a named revision and environment. It does not prove "not flaky".
9. **Redact secrets** (tokens, passwords, auth headers, cookies, connection strings) from every report, log excerpt and comment.

## Rationalizations to reject

| Excuse | Reality |
|--------|---------|
| "Tests pass, so requirements are met" | Tests prove what they assert. Map each criterion to a case. |
| "Just rerun it" | A rerun that passes is a data point, not a diagnosis. |
| "Works on my machine / locally" | Different environment is a hypothesis to test, not an answer. |
| "No errors in the logs" | Check the logs were captured and contain the time window. |
| "It's infra" | Show the infra signal (refused connection, OOM kill). Otherwise it is unexplained. |

## Final status line

DONE, DONE_WITH_CONCERNS and BLOCKED are defined in `~/.claude/skills/you-got-skills/skills/shared/completion-signals.md`; do not restate them. Automation-facing runs end with one JSON line and nothing after it. The harness only understands `status` of `DONE`, `PARTIAL` or `ERROR`, so carry the signal in a second key:

| Signal | `status` | Why |
|--------|----------|-----|
| DONE | `DONE` | Verified, evidence collected |
| DONE_WITH_CONCERNS | `DONE` | Finished; concerns are named in the report |
| BLOCKED | `ERROR` | Nothing trustworthy was produced; failing the job is correct, a green result would mislead |

`{"status":"DONE","signal":"DONE_WITH_CONCERNS","summary":"<one sentence>"}`, plus the generator's own keys: `scripts.flaky.report` adds `verdict`, `runs`, `failures`; `scripts.qa.report` adds `readiness`, `executed`, `failures`, `delta`. They print this line; copy it, do not rewrite it. If you must stop before a report generator can run (PR not found, NO-GO plan, loop gate failed, nothing identifiable), print `{"status":"ERROR","signal":"BLOCKED","summary":"<exact reason>"}` as the last line. `status` DONE only means the run finished: automation must read `readiness` (QA) or `verdict` (flaky) for the outcome.
