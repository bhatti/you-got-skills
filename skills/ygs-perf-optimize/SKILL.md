---
name: ygs-perf-optimize
description: Structured performance optimization — baseline, optimize one change at a time, prove improvement with statistical significance (Welch t-test), regression guard. Evidence required before and after.
argument-hint: "<what is slow> [--runs N] [--budget N]"
---

# Performance Optimize

For instrumentation design, read `~/.claude/skills/you-got-skills/skills/ygs-observe/SKILL.md`.
For test execution, read `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.

Never optimize without a measured baseline. Never accept an improvement without statistical significance. Never ship without a regression guard.

## Step 1: Define hypothesis

Before touching any code, answer:
1. **What is slow?** — Specific operation, endpoint, or code path (not "the app")
2. **What is the evidence?** — Profile output, metric, benchmark result, or user report with numbers
3. **What is the hypothesis?** — Why do you believe this specific change will improve it?
4. **What is the success criterion?** — Target: p95 < Xms, throughput > Y req/s, memory < Z MB

If you cannot answer all four, gather evidence first with `/ygs-observe` before optimizing.

## Step 2: Establish baseline

Run the benchmark N times (default: 10 runs, override with `--runs N`) and record:

| Metric | Value |
|--------|-------|
| Mean | |
| p50 | |
| p95 | |
| p99 | |
| Allocations (if measurable) | |
| Throughput (if applicable) | |

Use the language-appropriate benchmark tool:

| Language | Tool |
|----------|------|
| Rust | `cargo bench` (criterion) |
| Go | `go test -bench=. -benchmem -count=10` |
| Python | `pytest-benchmark` or `timeit` |
| Node/TS | `benchmark.js` or `autocannon` |

Save baseline results to `reports/perf-baseline-YYYY-MM-DD.txt`.

## Step 3: Optimization loop (budget: N iterations, default 5)

Each iteration:

1. **Propose one change** — the smallest possible modification targeting the hypothesis. No multi-change iterations.
2. **Apply the change**
3. **Measure** — same N runs as baseline
4. **Welch t-test** — test whether the improvement is statistically significant (p < 0.05):
   - If p ≥ 0.05: the change made no measurable difference — revert it
   - If p < 0.05 and mean improved: accept the change, record result
   - If p < 0.05 and mean worsened: revert immediately
5. **Log the iteration:**

```
Iteration N: <change description>
Before: mean=Xms, p95=Yms
After:  mean=Xms, p95=Yms
p-value: Z (significant: yes/no)
Decision: accepted | reverted
```

Stop the loop early if: budget exhausted, success criterion met, or two consecutive iterations produce no significant improvement (the remaining gains are likely noise).

**Statistical significance (p < 0.05):**
Use a Welch t-test on the two sample sets (before vs. after). Language tools:
- Python: `scipy.stats.ttest_ind(before, after, equal_var=False)` — check `pvalue < 0.05`
- R: `t.test(before, after, var.equal=FALSE)`
- Go/Rust/Node: use a statistics library or compute manually (Welch's formula: separate variance estimates per sample)

Do not eyeball "looks faster" as significance — noise in benchmark runs regularly exceeds 5-10% without statistical testing catching it.

## Step 4: Regression guard

After accepting changes, run the full test suite to verify correctness is preserved:

```bash
# via shared/test-runner.md
```

Then run benchmarks one final time and save to `reports/perf-final-YYYY-MM-DD.txt`.

If any test fails: revert the last accepted change and investigate before proceeding.

## Step 5: Write findings report

Write `reports/perf-YYYY-MM-DD.md` with:

```markdown
# Performance Optimization — <area>

**Date:** YYYY-MM-DD
**Hypothesis:** <what you expected to improve and why>
**Success criterion:** <target metric>

## Results

| Metric | Baseline | Final | Change |
|--------|----------|-------|--------|
| Mean | | | |
| p95 | | | |
| Allocations | | | |

## Changes applied

1. <change 1> — improved mean by X% (p=Y)
2. <change 2> — ...

## Changes rejected

1. <change> — no significant improvement (p=Z)

## Optimization budget

Budget: N iterations, used: M

## Conclusion

<Did we meet the success criterion? What is left? What should be investigated next?>
```

## Step 6: Completion

Report **DONE** with the before/after summary table.

Report **DONE_WITH_CONCERNS** if success criterion was not met — include the gap and next steps.

Suggest: `/ygs-observe` to instrument the optimized path for production monitoring.
