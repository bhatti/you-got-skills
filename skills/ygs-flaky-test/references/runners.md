# Runners: single-test commands and amplification flags

`scripts.flaky.detect_runner` builds these as argv lists. Use them by hand when the module is missing. Always disable result caches, or reruns replay the cached result. `npx --no-install` assumes a classic `node_modules`; with yarn PnP or pnpm use the project's own script (`yarn jest`, `pnpm exec jest`) as `--cmd`.

| Runner | Full file | Single test | Cache / shuffle / notes |
|--------|-----------|-------------|--------------------------|
| jest | `npx --no-install jest <file>` | `… -t "<name>"` | `--no-cache`; `--randomize` (jest 29.2+); `--runInBand` vs default workers changes timing |
| vitest | `npx --no-install vitest run <file>` | `… -t "<name>"` | `--sequence.shuffle --sequence.seed=<n>` |
| mocha | `npx --no-install mocha <file>` | `… --grep "<name>"` | `--parallel` to stress; `--retries 0` |
| playwright | `npx --no-install playwright test <file>` | `… -g "<name>"` | `--repeat-each=N`, `--workers=N`, `--retries=0` |
| pytest | `python3 -m pytest -p no:cacheprovider <file>` | `<file>::<Class>::<test>` or `-k "<name>"` | `-p randomly --randomly-seed=<n>` (pytest-randomly), `-n auto` (xdist) for stress, `-p no:randomly` to pin |
| go (`detect_runner`: `gotest`) | `go test -count=1 ./pkg` | `-run '^TestName$'` (subtests: `-run '^TestName$/^sub$'`) | `-count=1` is mandatory (no cache); `-shuffle=on`; `-race`; `-cpu 1,4`; `-parallel N` |
| maven | `mvn -q -Dtest=Class test` | `-Dtest=Class#method` | `-Dsurefire.runOrder=random`; `-DforkCount=N`; multi-module builds need `-pl <module> -DfailIfNoTests=false` |
| gradle | `./gradlew cleanTest test --tests Class` | `--tests Class.method` | `cleanTest` defeats the up-to-date check; `--parallel` |
| cargo | `cargo test --test <name>` | `cargo test <name>` (substring match; `--exact` needs the full module path) | `-- --test-threads=1` vs default; `--shuffle` on nightly |
| rspec | `bundle exec rspec <file>` | `… -e "<name>"` | `--order random --seed <n>` |
| dotnet | `dotnet test --no-restore` | `--filter "FullyQualifiedName~<name>"` | `-- RunConfiguration.MaxCpuCount=N` |

## Environment hygiene

- Reproduce on the revision named in the report (`git rev-parse HEAD`); run the install step the project documents (`npm ci`, `pip install -e .[test]`, `go mod download`) through `python3 -m scripts.common.env_scrub -- <cmd>` so repository install scripts never see the platform's tokens (`--keep NAME` for a variable the install genuinely needs).
- Services the test needs (database, queue): start them the way the repo documents (compose file, Makefile target) and check readiness by polling, not by sleeping.
- Platform matters: note OS/arch. A Linux-only failure will not show on macOS; say so under "Not judged".
- Limited CPU for constraint runs: `docker run --rm --cpus=1 -v "$PWD":/w -w /w <image> <cmd>` (requires a reachable Docker daemon; otherwise skip and say so).

## Unknown runner

Ask for `--cmd`. A Makefile target or script is fine. Provide `--isolate-cmd` with `{test}` when the runner can select one test by name; otherwise the loop runs the full command every time.
