# Evidence, duplicates and failure kinds

How the report keeps evidence readable and tells a product defect from trouble with the setup. Implemented in
`scripts/qa/dedup.py`, `verdict.py`, `evidence.py`.

## Duplicate findings (display-only)
- Cases that fail the same way (message equal after ids, timestamps, ports and durations are ignored; HTTP status codes
  stay distinct) form one group: count, affected cases, one sample. Failed and blocked cases with the same text are separate groups.
- Identical exchanges inside one case (a polling loop, a retry) are listed once with `xN`. The same exchange in
  **different** cases is not merged: an expected 401 is repeated legitimately and each case needs its own evidence.
- Long output is clipped to 1000 characters with the number of characters left out; copied text evidence is capped at 120000 characters.
- Grouping never changes a count, an outcome or the readiness. The table above still lists every case.

## Failure kinds
| kind | meaning | where |
| --- | --- | --- |
| `product` | the behaviour is wrong | counts against the change |
| `baseline` | baseline not ready / timed out | baseline side only; INCONCLUSIVE in the delta |
| `environment` | connection refused, 502-504, rate limit, cluster not ready | tagged in the report: rerun or fix the environment |
| `unverified` | the repro harness itself broke | tagged; nothing can be concluded |

Defaults live in `scripts/qa/verdict.py`; a project overrides a kind with `--failure-patterns file.json`
(`{"environment": ["regex", ...]}` replaces that kind's defaults) on `scripts.qa.delta` and `scripts.qa.report`.

## Evidence files
`reports/evidence/` is deleted and rebuilt on every run, so reruns never leave `name-2.png` copies. Only files under the
evidence roots are copied; text is redacted. `--embed-evidence` inlines images up to 300 KB each (3 MB total) into
`report.html` only; `report.md` keeps links.
