---
name: ygs-investigate
description: Disciplined debugging — build feedback loop first, reproduce, hypothesize (3-5 ranked), instrument, fix, regression-test. No fixes without root cause.
---

# Investigate

**Iron law: No fixes without root cause. No hypotheses without a feedback loop.**

## Phase 1: Build a feedback loop

**This is the skill.** Everything else is mechanical. A fast, deterministic pass/fail signal is the leverage point. Spend disproportionate effort here.

Ways to construct one (try in order, escalating effort):
1. **Failing test** at whatever seam reaches the bug
2. **CLI invocation** with fixture input, diffing output against known-good
3. **HTTP script** (curl/httpie) against a running dev server
4. **Headless browser** — Playwright/Puppeteer script for UI-visible bugs
5. **Replay captured trace** — save real request/event, replay through the code path
6. **Throwaway harness** — minimal subset of system exercising the bug path
7. **Property/fuzz loop** — 1000 random inputs looking for failure mode
8. **Bisection harness** — automate "build at commit X, check" for `git bisect run`
9. **Differential testing** — same input through old version vs new, diff outputs
10. **HITL (Human-in-the-loop)** — when the signal requires human judgment (visual glitch, timing-sensitive), script the setup and have user confirm pass/fail

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

## Phase 7: Architectural handoff

If the bug revealed structural problems, flag them explicitly:

- **Missing test seam** — the correct place to assert wasn't reachable through any public interface
- **Tangled callers** — N callers cross-cutting through the bug's module suggest the module is too shallow
- **Implicit coupling** — the bug existed because two modules shared assumptions not enforced by an interface
- **Missing invariant** — a state that should be impossible was representable

These are not things to fix now — they are findings for architecture review. Suggest `/ygs-refine-architecture` if any apply.

## Completion

Report **DONE** with:
- **Root cause:** One sentence explaining why
- **Fix:** What was changed
- **Regression test:** Where it lives
- **Prevention:** What would have caught this earlier (architectural gap? missing test seam?)
- **Architectural findings:** (if any) Structural issues revealed by this bug

Or **BLOCKED** if root cause cannot be determined (explain what was tried and what would unblock).

---

## CI/Pipeline failure specifics

When the bug is a CI/pipeline failure rather than a code bug, apply the same phases above with these additional heuristics:

**Phase 1 adaptation — identify the root cause stage:**
- Find the **earliest** failing stage — downstream failures are cascade effects, not root causes
- Differentiate signal from noise: test names containing "failed", expected error output in logs, property dumps are NOT failures

**Log noise filtering:**
- Ignore lines that are test names referencing failure scenarios (e.g., `"should fail when..."`)
- Ignore expected error output from negative test cases
- Ignore framework stack traces from passing assertion libraries
- Focus on: unexpected exit codes, assertion failures with actual vs expected, timeout messages, OOM kills

**Flaky test detection:**
- If the same test passes on retry without code changes, it's flaky
- Common flaky causes: timing dependencies, shared mutable test state, network calls in unit tests, filesystem ordering assumptions
- Flag as flaky, don't chase as a real bug unless user confirms it's consistent
