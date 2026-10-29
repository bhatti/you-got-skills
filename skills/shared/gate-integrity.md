# Gate Integrity

Shared by: `ygs-code-review`, `ygs-review-pr`, `ygs-review-deep`, `ygs-review-ready`, `ygs-ship`

A change that makes its own proof weaker is the most expensive kind of green: CI passes because the check was loosened, not because the code works. For every hunk in the diff ask one question: **does this make a failure stop being reported?** If yes, it is a finding.

## Patterns to look for

| Pattern | Examples |
|---------|----------|
| Test removed or disabled | deleted test or file, `.skip`, `xit`, `@Ignore`, `t.Skip()`, `pytest.mark.skip`, a case commented out; a snapshot or golden file regenerated without a test that explains why |
| Assertion weakened | exact value replaced by a bound or `non-null`; `assertEqual` replaced by `assertTrue(x)`; a loop of assertions replaced by one on the first element |
| Threshold or budget relaxed | coverage floor, timeout, retry count, latency or size limit, lint severity, flaky-retry count raised |
| Suppression added | `eslint-disable`, `@ts-ignore`, `# noqa`, `# type: ignore`, `#nosec`, `nolint`, `nosemgrep`, `allow(...)` without a stated reason |
| Pipeline bypass | `[skip ci]`, `--no-verify`, `continue-on-error`, `|| true` after a check, a required job made optional, a path filter that now excludes the changed code |
| Failure swallowed | empty `catch`, `except: pass`, an error logged and then ignored, a default returned in place of the error |
| Fake proof | placeholder or stub committed where the feature should be; a mock of the unit under test; a test whose expected value is computed the way the code computes it; production code that detects the test environment |
| Goalposts moved | acceptance criteria, the definition of done, or the issue text edited to match what was delivered |

## Rules

- **One question per hunk,** answered from the diff, not from the commit message. A message that says "flaky" or "fixes CI" does not remove the finding.
- **These findings are ASK, never auto-fixed.** State the hunk (`file:line`), what failure it silences, and what evidence would justify it. The author decides.
- **Tightening is silent; loosening is loud.** Raising a threshold, adding an assertion or enabling a rule needs no comment. Anything in the table above needs a stated reason in the PR.
- **Prefer a ratchet to a number.** A floor should hold today's value, not a round number that was already exceeded.
- **If the check cannot run, say so.** A gate scan that failed to execute is "not checked", never "clean".

## Severity

Deleted or skipped tests, a swallowed failure on a changed path, and a pipeline bypass are SHOULD at minimum (MUST when the code they guard handles money, auth or data loss). A new suppression with a reason and a ticket is MAY.
