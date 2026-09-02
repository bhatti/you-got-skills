---
name: ygs-triage-ci
description: CI/pipeline failure investigation — classify failure type, reproduce locally, fix or escalate with a structured brief. Faster than ygs-investigate for CI-specific failures.
argument-hint: "<build-url-or-run-id> [--branch <branch>]"
---

# Triage CI

For test execution patterns, read `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.

Use when a CI build has failed and you need to understand why quickly — without starting a full debugging session. For failures with a known repro outside CI, use `/ygs-investigate` instead.

## Step 1: Detect platform and fetch failure logs

Determine the CI platform from the URL or project config:

| Platform | Fetch logs |
|----------|-----------|
| GitHub Actions | `gh run view <run-id> --log-failed` |
| Jenkins | Jenkins REST API: `GET <jenkins-url>/job/<path>/<build>/api/json` for status; `GET .../consoleText` for full log; or use the project's Jenkins CLI wrapper if one exists |
| Bitbucket Pipelines | Bitbucket REST API: `GET /pipelines/{uuid}/steps/{step-uuid}/log` |
| Generic | Ask user for log output |

Collect: failure summary, failing step name, raw log tail (last 200 lines).

## Step 2: Classify the failure type

| Type | Signals |
|------|---------|
| **Test failure** | Test framework output (FAIL, FAILED, AssertionError, panic) |
| **Build error** | Compiler or linker error, missing dependency, import error |
| **Infra / timeout** | OOMKilled, connection refused, timeout waiting for service, disk full |
| **Flaky** | Passes on re-run, timing-sensitive output, non-deterministic ordering |

State the type before proceeding — the fix path differs per type.

## Step 3: Root-cause per failure type

### Test failure
1. Extract failing test names from the log
2. Find the last commit where these tests passed:
   ```bash
   git log --oneline -20
   ```
3. Identify the diff most likely responsible:
   ```bash
   git diff HEAD~1 -- <relevant-files>
   ```
4. Reproduce locally using `shared/test-runner.md` before proposing a fix

### Build error
1. Extract the exact compiler/linker error with file:line
2. Read the referenced file at that location
3. Trace the error to its root cause (missing import, type mismatch, undefined symbol)
4. Check whether a recent dependency version change is involved

### Infra / timeout
1. Check for resource limits (OOMKilled → memory; timeout → slow test or missing service)
2. For service-dependency timeouts: check whether the service starts in CI (look for setup steps in the config)
3. For disk/memory: identify the source (large test artifacts, unbounded allocation, log flooding)
4. This is usually not fixable in code — escalate with a structured brief (see Step 5)

### Flaky test
1. Check run history: does this failure appear intermittently across unrelated commits?
   ```bash
   gh run list --workflow=<name> --branch=main --limit=20
   ```
2. Look for timing patterns (`sleep`, fixed port, wall-clock dependency, non-deterministic map/set ordering)
3. Look for shared mutable state between tests
4. If confirmed flaky: skip the test at the framework level (e.g., `@pytest.mark.skip`, `#[ignore]`, `it.skip()`, `t.Skip()`), file a task to fix the root cause, and re-run CI to unblock

## Step 4: Fix (test failures and build errors only)

Apply the minimum fix that restores CI:
- Do not refactor unrelated code while fixing
- Write a regression test for the exact failure scenario
- Run tests locally to confirm the fix before reporting

For infra and confirmed flaky failures, skip this step and go directly to Step 5.

## Step 5: Structured brief

Report with:
- **Failure type:** test failure | build error | infra | flaky
- **Root cause:** one sentence — what specifically failed and why
- **Evidence:** file:line or log excerpt that proves it
- **Fix applied / proposed:** what was changed or what should change
- **Regression test:** added yes/no
- **Reproducible locally:** yes / no / not applicable
- **Escalation needed:** yes (infra, flaky) / no

Report **DONE** or **DONE_WITH_CONCERNS** (if fix is uncertain or escalation is needed).

Suggest: `/ygs-investigate` for root causes that need deeper debugging.
