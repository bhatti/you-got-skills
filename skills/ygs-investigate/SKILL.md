---
name: ygs-investigate
description: Disciplined debugging — build feedback loop first, reproduce, hypothesize (3-5 ranked), instrument, fix, regression-test. No fixes without root cause.
---

# Investigate

**Iron law: No fixes without root cause. No hypotheses without a feedback loop.**

## Phase 1: Build a feedback loop

**This is the skill.** Everything else is mechanical. A fast, deterministic pass/fail signal is the leverage point. Spend disproportionate effort here.

Ways to construct one (try in order):
1. **Failing test** at whatever seam reaches the bug
2. **CLI invocation** with fixture input, diffing output against known-good
3. **HTTP script** (curl/httpie) against a running dev server
4. **Replay captured trace** — save real request/event, replay through the code path
5. **Throwaway harness** — minimal subset of system exercising the bug path
6. **Property/fuzz loop** — 1000 random inputs looking for failure mode
7. **Bisection harness** — automate "build at commit X, check" for `git bisect run`

### Iterate on the loop
- Can you make it faster? (Cache setup, narrow scope)
- Can you make the signal sharper? (Assert on specific symptom, not "didn't crash")
- Can you make it more deterministic? (Pin time, seed RNG, isolate filesystem)

### When you cannot build a loop
Stop and say so explicitly. List what you tried. Ask user for: captured artifact (logs, HAR file, recording), access to reproducing environment, or permission to add instrumentation. **Do not proceed without a loop.**

## Phase 2: Reproduce

Run the loop. Confirm:
- The failure matches what the **user** described (not a different nearby failure)
- Reproducible across multiple runs
- Exact symptom captured (error message, wrong output, timing)

## Phase 3: Hypothesize

Generate **3-5 ranked hypotheses** before testing any of them. Single-hypothesis thinking anchors on the first plausible idea.

Each hypothesis must be **falsifiable**:
> "If <X> is the cause, then <changing Y> will make the bug disappear / <changing Z> will make it worse."

If you cannot state the prediction, the hypothesis is a vibe — discard or sharpen it.

Show the ranked list to the user before testing. They often have domain knowledge that re-ranks instantly.

## Phase 4: Instrument

Each probe maps to a specific prediction from Phase 3. **Change one variable at a time.**

Preference:
1. Debugger/REPL inspection (one breakpoint beats ten logs)
2. Targeted logs at boundaries that distinguish hypotheses
3. Never "log everything and grep"

Tag every debug artifact with unique prefix (e.g. `[DEBUG-x7k2]`) for cleanup.

## Phase 5: Fix + regression test

1. Write regression test at the correct seam **before** the fix
2. Watch it fail
3. Apply minimal fix addressing root cause
4. Watch it pass
5. Re-run the Phase 1 feedback loop against original scenario
6. Check for same bug class elsewhere in codebase

If no correct test seam exists, **that itself is a finding** — flag the architectural gap.

## Phase 6: Cleanup

- [ ] Original feedback loop no longer reproduces the bug
- [ ] Regression test passes
- [ ] All `[DEBUG-...]` instrumentation removed
- [ ] Throwaway harnesses deleted
- [ ] Root cause stated in commit message

## Completion

Report **DONE** with:
- **Root cause:** One sentence explaining why
- **Fix:** What was changed
- **Regression test:** Where it lives
- **Prevention:** What would have caught this earlier (architectural gap? missing test seam?)

Or **BLOCKED** if root cause cannot be determined (explain what was tried and what would unblock).
