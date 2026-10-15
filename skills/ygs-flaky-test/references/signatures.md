# Flakiness signatures and what to do about them

Hypotheses, not verdicts. Each row: what you see, why it flakes, how to test the hypothesis, and the fix to recommend.

| Signature | Why it flakes | Test the hypothesis | Recommend |
|-----------|---------------|---------------------|-----------|
| Fixed sleep / `setTimeout` then assert | Work takes longer on a slow or loaded machine | `--parallel` stress or `--cpus=1`; failure rate rises | Poll for the condition with a deadline, or await the event |
| Timeout just above an internal retry budget | Condition never satisfied inside the window | Log when the condition becomes true; compare to the limit | Wait on the condition; raise limits only with measurements |
| Fixed port / `EADDRINUSE` | Parallel runs or leftover processes hold the port | Run two copies at once; `lsof -i :<port>` | Bind port 0, pass the port to the client |
| Fails only in full-file/suite runs | Another test leaves state behind | `bisect_polluter` | Isolate the shared state; reset in setup/teardown |
| Wall clock (`now()`, `Date.now()`) in assertions | Result depends on time of day, DST, speed | Vary `TZ`; freeze the clock | Inject a clock / fake timers |
| Hard-coded future dates, expiring certificates | Time bomb: passes until the date passes | Set the clock past the date | Compute relative dates; generate fixtures at test time |
| Unseeded random / `uuid4` in expected values | Inputs differ per run | Log and replay the seed | Seed it, or assert properties instead of values |
| Unordered iteration (map/set/keys) | No guaranteed order | Shuffle; many runs with hash seeds (`PYTHONHASHSEED`) | Sort before comparing |
| Shared temp path or fixed file name | Concurrent tests overwrite each other | `--parallel`; look for leftover files | Per-test temp dir |
| Real network or third-party service | Latency, rate limits, outages | Run offline | Stub or record the dependency |
| Shared environment variables / globals | Leaks between tests | Run tests in reverse order | Save and restore, or per-test copies |
| Async step not awaited | Assertion runs before the work finishes | Add a log after the awaited condition; compare ordering | Await every step; assert after the condition |
| Data race / deadlock | Unsynchronized shared memory | `go test -race`, thread sanitizers, `--cpus` variations | Synchronize; join workers before asserting |
| OOM kill, too many open files | Resource pressure under load | `dmesg`, container memory stats during runs | Fix the leak; lower parallelism; give the test enough resources |
| Existing retries / flaky markers | Hide the real failure rate | Run with retries disabled | Measure first; remove masking when fixed |

## Weak tests that look flaky

- **Mirror assertions:** the expected value is computed by the same logic as the code under test, so failures are noise.
- **Change detectors:** exact error wording or generated ids asserted.
- **Source grepping:** tests that read the implementation instead of running it.

When a test cannot fail for the right reason, say it. "Which production change would make this test fail?" is the question to answer.

## Bounded investigation

One hypothesis at a time, each with a falsifying observation. After 3 failed hypotheses stop and report the unexplained flake with the list of what was checked and ruled out.
