---
name: ygs-build-optimize
description: CI/CD build optimization — profile the pipeline, identify bottlenecks, apply targeted fixes (caching, parallelism, image layers, sparse checkout). Evidence-driven, one change at a time.
argument-hint: "[--pipeline <name-or-url>] [--budget <minutes>] [--focus cache|parallel|image|all]"
---

# Build Optimize

For test execution, read `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.
For performance measurement, read `~/.claude/skills/you-got-skills/skills/ygs-perf-optimize/SKILL.md` (the baseline/measure/prove cycle applies here too).

Use when CI is slow and you want to reduce pipeline duration. For running only affected tests, use `/ygs-test-impact`. For investigating CI failures, use `/ygs-triage-ci`.

## Step 1: Profile the pipeline

Before optimizing anything, measure the current state:

### Collect timing data

| Platform | How to get step timings |
|----------|------------------------|
| GitHub Actions | `gh run view <id> --json jobs --jq '.jobs[] \| {name, startedAt, completedAt}'` |
| Jenkins | `GET <url>/job/<name>/<build>/wfapi/describe` (Pipeline Stage View API) |
| GitLab CI | `GET /api/v4/projects/:id/pipelines/:id/jobs` |
| Formicary | Check task durations in job execution dashboard |
| Local | Wrap each step with `time` or `date +%s` markers |

### Build the waterfall

List every pipeline step with:
- Step name
- Duration (seconds)
- Sequential or parallel with other steps
- Cache hit/miss (if applicable)

Calculate:
- **Critical path:** longest sequential chain (this is what determines wall-clock time)
- **Utilization:** sum of all step durations / (critical path x max parallelism)
- **Top 3 bottlenecks:** longest steps on the critical path

### Classify bottlenecks

| Category | Signals | Typical fix |
|----------|---------|-------------|
| **Dependency install** | npm install > 60s, pip install > 30s | Layer caching, lock-file hash key |
| **Compilation** | tsc/gcc/javac > 120s | Incremental builds, build cache, type-free lint |
| **Test execution** | Tests > 5min | Sharding (see `/ygs-test-impact`), parallelism |
| **Docker build** | docker build > 3min | Multi-stage, layer ordering, BuildKit cache |
| **Checkout** | git clone > 30s | Sparse checkout, shallow clone |
| **Setup** | Base image pull > 60s | Pre-built setup images, cached layers |
| **Artifact transfer** | Upload/download > 60s | Compression, selective artifacts |

## Step 2: Apply fixes (one at a time)

Pick the single largest bottleneck on the critical path. Apply the appropriate fix. Measure. Repeat.

### Fix: Dependency caching

```yaml
# GitHub Actions pattern
- uses: actions/cache@v4
  with:
    path: ~/.npm  # or ~/.cache/pip, ~/.m2, ~/.gradle
    key: deps-${{ hashFiles('**/package-lock.json') }}
    restore-keys: deps-
```

For formicary/k8s: use workspace volumes that persist across tasks, or artifact caching between job runs.

### Fix: Parallel test sharding

Split tests into N shards running on parallel workers:

```yaml
# GitHub Actions matrix
strategy:
  matrix:
    shard: [0, 1, 2, 3, 4, 5, 6, 7]
```

```yaml
# Formicary fan-out
fan_out:
  source: TestShards
  item_var: shard
  max_parallel: 8
```

Optimal shard count: start with CPU count of your CI runner, then adjust based on shard balance (see `/ygs-test-impact` Step 3).

### Fix: Type-free linting

Separate type checking from linting — lint rules that don't need types run 3-5x faster:

```bash
# TypeScript: eslint without type-aware rules
eslint --no-eslintrc -c .eslintrc.fast.json .

# Or use biome/oxlint for type-free lint (10-100x faster than eslint)
biome check .
```

Run type checking as a separate parallel step.

### Fix: Sparse checkout

For monorepos where most CI runs touch a small subset:

```bash
git clone --filter=blob:none --sparse <repo>
git sparse-checkout set <paths-relevant-to-this-PR>
```

Speedup depends on repo size — Linear reported 94s to 20s (79% reduction) for their monorepo.

### Fix: Pre-built setup images

Build a CI-specific image with dependencies pre-installed:

```dockerfile
FROM node:20-slim
COPY package-lock.json .
RUN npm ci --ignore-scripts
# Now the CI step just copies source and runs tests
```

Rebuild weekly or on lock-file change. Linear reported 44-73s to 16-18s (63-75% reduction).

### Fix: Docker build optimization

```dockerfile
# Order layers by change frequency (least to most)
COPY package-lock.json .
RUN npm ci
COPY . .
RUN npm run build
```

Enable BuildKit:
```bash
DOCKER_BUILDKIT=1 docker build --cache-from=type=registry,ref=<cache-image> .
```

### Fix: Incremental compilation

```bash
# TypeScript: use project references + tsc --build
tsc --build --incremental

# Rust: sccache
RUSTC_WRAPPER=sccache cargo build

# Go: default build cache (ensure GOPATH/pkg persists)
```

## Step 3: Measure improvement

After each fix, run the pipeline 3 times and record:

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Critical path (s) | | | |
| Total duration (s) | | | |
| Cache hit rate (%) | | | |
| Step X duration (s) | | | |

Only proceed to the next optimization if:
1. The improvement is > 10% for the targeted step
2. No regressions in other steps
3. Tests still pass

## Step 4: Verify no regressions

Run the full test suite after all optimizations to confirm:
- All tests still pass
- No flaky tests introduced by parallelism changes
- Cache invalidation works correctly (modify a dependency, verify rebuild)

## Step 5: Report

- **Pipeline profile:** waterfall with step timings (before/after)
- **Optimizations applied:** list with measured improvement per fix
- **Total improvement:** wall-clock reduction percentage
- **Remaining bottlenecks:** what to tackle next
- **Risks:** any caching edge cases, parallelism race conditions

Report **DONE** with the total speedup achieved.
Report **DONE_WITH_CONCERNS** if:
- Cache invalidation behavior is uncertain
- Parallelism introduced flaky test risk
- Optimization budget (`--budget`) exhausted before all bottlenecks addressed

Suggest: `/ygs-test-impact` for test-level optimization, `/ygs-perf-optimize` for application-level performance.
