---
name: ygs-spike
description: Time-boxed spike to validate a hypothesis — quick-and-dirty implementation to prove feasibility, benchmark performance, or test integration. Not production code. Use for risky unknowns before committing to a full design.
---

# Spike

A spike is a time-boxed experiment to reduce risk. It answers a specific question — "Can we do X?" / "How fast is Y?" / "Does Z integrate cleanly?" — with minimal throwaway code. Not production quality. Not full coverage. Just enough to decide.

## Step 1: Define the hypothesis

Ask the user (or infer from context):
- **Question:** What are we trying to learn?
- **Success criteria:** What result would prove/disprove the hypothesis?
- **Time box:** How long before we stop regardless? (default: 2-4 hours)
- **Branch:** Work on a `spike/` or `fafo/` branch (not main)

Examples:
- "Can our DB handle 10K writes/sec with this schema?" → benchmark
- "Does this library integrate with our auth stack?" → integration test
- "Is the algorithm fast enough for real-time?" → performance proof
- "Can we parse this format without a custom grammar?" → feasibility

## Step 2: Scope what's explicitly allowed

Spikes deliberately relax production standards:
- ✅ Hardcoded values, magic numbers
- ✅ Minimal error handling (panic/unwrap/crash is fine)
- ✅ No comprehensive tests (one happy-path test to prove it works)
- ✅ Copy-paste from existing code without refactoring
- ✅ Skip documentation, skip linting
- ✅ Use `todo!()`, `unimplemented!()`, stub interfaces
- ✅ Print/log debugging instead of structured observability

What's still NOT allowed:
- ❌ Modifying production code on main branch
- ❌ Committing secrets or credentials
- ❌ Breaking existing tests (spike lives in isolation)
- ❌ Spending time beyond the time box without checking in

## Step 3: Explore existing code

```bash
# Find relevant existing implementations to base spike on
find . -name "*.rs" -o -name "*.ts" -o -name "*.py" -o -name "*.go" | head -20
```

Read existing code that's relevant to the hypothesis. Spikes build on what exists — don't start from zero if there's prior art in the codebase.

## Step 4: Build the minimum viable experiment

Focus ruthlessly on the hypothesis:
- If benchmarking: write the hot path + measurement harness only
- If testing integration: wire the two systems + one end-to-end call
- If proving feasibility: implement the core algorithm + one example input
- If testing scalability: simulate load at target scale

### Measurement (if performance spike)

```
- Baseline: [current measurement or "N/A"]
- Target: [what would prove the hypothesis]
- Method: [how we're measuring — criterion, hyperfine, manual timing, load test tool]
```

## Step 5: Verify the result

Before recording findings, confirm the spike actually proved what you think it proved:
- Does the test exercise the real constraint, or a simplified version that might not transfer?
- Could the result be an artifact of small scale, test data, or missing error paths?
- Would the approach survive real-world conditions (concurrent access, large data, network partitions)?
- Is "it compiled and ran" the same as "it works for our use case"?

CONFIRMED means you have evidence. INCONCLUSIVE means you ran out of time. Never mark CONFIRMED if the experiment was too narrow to be conclusive.

## Step 6: Record findings

Create `docs/spikes/YYYY-MM-DD-<slug>.md` with:

```markdown
# Spike: [Title]

**Date:** YYYY-MM-DD
**Time spent:** [actual]
**Hypothesis:** [what we were testing]
**Result:** CONFIRMED | REFUTED | INCONCLUSIVE

## What we learned
<!-- Key findings, measurements, surprises -->

## Evidence
<!-- Benchmark results, logs, screenshots, test output -->

## Decision
<!-- Based on findings: proceed with full implementation / pivot / abandon -->

## If proceeding
<!-- What changes from spike to production: error handling, tests, architecture adjustments -->
```

## Step 7: Completion

Report **DONE** with:
- Hypothesis: confirmed / refuted / inconclusive
- Key evidence (numbers, outputs)
- Recommendation: proceed / pivot / abandon
- If proceeding: what the production implementation needs beyond the spike
- What the spike did NOT test that production will need to handle

Suggest: `/ygs-refine-trd` if proceeding to design, or `/ygs-retro` if abandoning.
