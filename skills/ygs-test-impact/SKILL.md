---
name: ygs-test-impact
description: Test impact analysis — map changed files to affected tests, partition into balanced shards, run only what matters. Reduces CI time by 60-90% on typical PRs.
argument-hint: "<pr-number-or-branch> [--base <branch>] [--shards N] [--strategy naming|imports|both]"
---

# Test Impact Analysis

For test execution patterns, read `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.

Use when CI is slow and you want to run only the tests affected by a change — not the full suite. For full CI optimization (caching, parallelism, image layers, sparse checkout), use `/ygs-build-optimize` instead.

## Step 1: Identify changed files

Determine the set of files changed relative to the base branch:

```bash
# From PR number
gh pr diff <pr-number> --name-only

# From branch
git diff --name-only origin/<base>...HEAD
```

Classify each file by type:

| Type | Pattern | Impact |
|------|---------|--------|
| **Source** | `src/`, `lib/`, `app/` | Map to tests via naming + imports |
| **Test** | `test/`, `tests/`, `*_test.*`, `*.spec.*` | Run directly |
| **Config** | `*.yaml`, `*.json`, `*.toml`, `Dockerfile` | Run full suite (broad impact) |
| **Docs** | `*.md`, `docs/` | Skip tests unless doc-testing is configured |
| **Build** | `Makefile`, `build.gradle`, `package.json` (deps changed) | Run full suite |

If ANY config or build file changed, recommend running the full suite — test impact analysis cannot safely narrow the scope.

## Step 2: Map source files to affected tests

Apply two strategies (use `--strategy` to select, default `both`):

### Strategy 1: Naming convention mapping

Map source files to test files by naming convention:

| Source | Test candidate |
|--------|---------------|
| `src/auth/login.py` | `tests/auth/test_login.py`, `tests/test_auth_login.py` |
| `src/auth/login.ts` | `src/auth/__tests__/login.test.ts`, `src/auth/login.spec.ts` |
| `src/auth/login.go` | `src/auth/login_test.go` (same package — Go convention) |
| `src/auth/Login.java` | `src/test/java/.../LoginTest.java` (Maven layout) |
| `src/auth/Login.kt` | `src/test/kotlin/.../LoginTest.kt` (Gradle/Maven layout) |
| `lib/auth/login.rb` | `spec/auth/login_spec.rb`, `test/auth/login_test.rb` |
| `src/Auth/Login.cs` | `src/Auth.Tests/LoginTests.cs` (.NET convention) |
| `crates/auth/src/login.rs` | `crates/auth/tests/login.rs` + inline `#[cfg(test)]` |

For each source file, check if the candidate test file exists. If not, log it as unmapped.

### Strategy 2: Import graph analysis

For languages with static imports, trace reverse dependencies:

```bash
# Python: grep for imports of the changed module
grep -rl "from auth.login import\|import auth.login" tests/

# TypeScript/JavaScript: grep for imports
grep -rl "from ['\"].*auth/login['\"]" --include="*.test.*" --include="*.spec.*" src/ tests/

# Go: grep for package imports (same-package tests are implicit dependents)
grep -rl '"github.com/org/repo/auth"' --include="*_test.go" .

# Java/Kotlin: grep for imports (fully-qualified class names)
grep -rl "import.*auth\.Login" --include="*Test.java" --include="*Test.kt" src/test/

# C#: grep for using statements
grep -rl "using.*Auth\.Login" --include="*Tests.cs" .

# Ruby: grep for require
grep -rl "require.*auth/login" --include="*_spec.rb" --include="*_test.rb" spec/ test/
```

**Strongly-typed language considerations:**
- **Rust**: Inline `#[cfg(test)]` modules mean the source file IS also a test file — always include changed `.rs` files that contain `#[cfg(test)]`
- **Go**: Tests live in the same package (same directory) — any change to a `.go` file affects all `_test.go` files in the same directory
- **Java/Kotlin**: Inner classes and package-private access mean tests may depend on classes without explicit imports — include same-package tests as safety net
- **C#**: Partial classes can span files — if one partial is changed, tests for any partial are affected

Combine results from both strategies. Deduplicate.

### Unmapped files

If a source file has no mapped tests:
1. Log it as a coverage gap
2. Check if the file is tested indirectly (called by a tested module)
3. If uncertain, include the parent directory's test suite as a safety net

## Step 3: Partition into balanced shards

Sort affected tests by estimated duration (use historical timing data if available, otherwise file size as proxy).

Apply greedy bin-packing (longest-processing-time-first):

1. Create N empty shards (default: CPU count, override with `--shards`)
2. For each test (sorted by duration descending):
   - Assign to the shard with the smallest total duration
3. Output: JSON array of shards, each with test paths and estimated duration

```json
[
  {"shard_id": 0, "tests": ["tests/auth/test_login.py", "tests/auth/test_session.py"], "estimated_s": 45},
  {"shard_id": 1, "tests": ["tests/api/test_endpoints.py"], "estimated_s": 42}
]
```

Verify shard balance: the ratio of max_shard_duration to min_shard_duration should be < 1.5. If worse, re-partition with more shards.

## Step 4: Execute shards

Run each shard using the project's test runner (detect from `shared/test-runner.md`):

```bash
# Python
pytest <test-files> --junit-xml=shard_N.xml

# JavaScript/TypeScript
jest --testPathPattern="<test-files-pipe-separated>"

# Go
go test <packages>

# Java/Gradle
gradle test --tests "<test-classes>"
```

If running in CI (formicary), each shard is a separate fan-out task — they run in parallel across workers.

If running locally, run shards sequentially or use `parallel` / `xargs -P`.

## Step 5: Report

Produce a structured report:

- **Changed files:** N source, N test, N config, N docs
- **Affected tests:** N tests mapped (N via naming, N via imports, N overlap)
- **Unmapped files:** list (coverage gaps)
- **Shards:** N shards, estimated wall clock M seconds (vs S seconds full suite)
- **Speedup:** X% reduction vs full suite
- **Results:** N passed, N failed, N skipped per shard

If any tests failed, report the failures with file:line and suggest `/ygs-triage-ci` for investigation.

Report **DONE** if all affected tests pass.
Report **DONE_WITH_CONCERNS** if:
- Unmapped files exist (potential coverage gap)
- Config/build files changed (full suite recommended)
- Shard balance ratio > 2.0 (poor parallelization)

Suggest: `/ygs-build-optimize` for broader CI optimization beyond test selection.
