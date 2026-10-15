# Regression mode: baseline vs candidate

Goal: show what the change broke or fixed by running the **same cases** on the baseline and the candidate. One environment at a time, one wrapper per side (`qa-out/run-all.sh` runs the API script and the UI runner together, see `environments.md`).

## Flow

```bash
for s in baseline candidate; do [ -f qa-out/$s/env-state.json ] && python3 -m scripts.qa.env_manager cleanup --state qa-out/$s/env-state.json; done   # leftovers of a killed run
rm -rf qa-out/baseline qa-out/candidate
# 1. baseline first, so a bad baseline is known before the candidate is built
python3 -m scripts.qa.env_manager run --kind <k> <baseline flags> --side baseline  --out-dir qa-out -- bash qa-out/run-all.sh
# 2. candidate: same command, candidate flags
python3 -m scripts.qa.env_manager run --kind <k> <candidate flags> --side candidate --out-dir qa-out -- bash qa-out/run-all.sh
# 3. compare
python3 -m scripts.qa.delta --plan qa-out/plan.json --baseline qa-out/baseline/results.jsonl --candidate qa-out/candidate/results.jsonl \
  --baseline-log qa-out/baseline/logs/env.log --candidate-log qa-out/candidate/logs/env.log --out qa-out/delta.json
# 4. report
python3 -m scripts.qa.report --mode regression --plan qa-out/plan.json --candidate qa-out/candidate/results.jsonl --delta qa-out/delta.json \
  --pr qa-out/pr-context.json --findings qa-out/findings.json --target-url <url> --baseline-url <url> \
  --env-failure qa-out/baseline/env-failure.json --env-failure qa-out/candidate/env-failure.json --out-dir reports
```

Keep the same ports, data and credentials on both sides. The report is BLOCKED in regression mode when no delta was produced.

## Delta classes

| Baseline | Candidate | Delta | Meaning |
|----------|-----------|-------|---------|
| pass | pass | BOTH_PASS | no change |
| pass | fail | **REGRESSION** | the change broke it |
| fail | pass | FIXED | the change fixed it |
| fail | fail | PRE_EXISTING | broken before this PR; report, do not blame (readiness is still Not ready: the candidate has failing cases) |
| none | any | NEW | case only exists on the candidate |
| any | none | REMOVED | case only exists on the baseline |
| blocked/infra on either side | | INFRA | not comparable; never a regression |
| not-run/inconclusive | | INCONCLUSIVE | not comparable |

Run status, in precedence: **BLOCKED** (a side executed no planned case) > **REGRESSION** > **NEW_FAILURES** (a candidate-only case fails) > **INCOMPLETE** (any INFRA, INCONCLUSIVE or REMOVED case) > **PRE_EXISTING** (failures on both sides) > **FIXED** > **CLEAN**. CLEAN means every case passes on both sides.

## Reading the result

- A REGRESSION needs evidence on both sides: the baseline pass output and the candidate failure output. Show both (the report links them).
- A case that "fixes" something the PR was meant to fix is the expected FIXED: say which criterion it supports.
- A REGRESSION in an area the diff did not touch suggests environment drift or flakiness: rerun that case on both sides before reporting it. If it flips, report it as inconclusive and point to `/ygs-flaky-test`.
- If baseline setup failed, say so; do not report the candidate as "no regressions".

## Log comparison

`scripts.qa.delta` clusters ERROR/WARN lines (numbers, ids, timestamps and IPs normalized, except HTTP status codes; a JSON `level` field outranks keywords; DEBUG/INFO/TRACE lines are ignored) and lists clusters that are new on the candidate or at least three times as frequent (and at least five occurrences). Known harmless noise goes in a file passed with `--noise` (one regex per line). Those patterns are a concern to read, not proof of a bug.

## Visual comparison

`SMOKE*` rows are listed in the delta but never count toward the plan's verdict. Screenshots of the same case on both sides are linked from the delta table. Look at both; describe differences in terms of content and layout. Do not run pixel diffs or claim visual equality.

## Performance (optional, `--perf`)

p50 per endpoint on each side (see `api-testing.md`); pass `--perf-baseline perf-baseline.json --perf-candidate perf-candidate.json` to `scripts.qa.delta`. Flag a candidate p50 more than 20% above the baseline. Both sides must run on the same machine and data volume; otherwise say the numbers are not comparable.
