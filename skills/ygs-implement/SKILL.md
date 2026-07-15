---
name: ygs-implement
description: Implement a task from the backlog with execution discipline — scope guardrails, checkpoints, deviation logging. Challenges premises, verifies against code reality, proves correctness by running it.
---

# Implement

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — you own the result.
For detailed principles, read `references/implementation-principles.md` (functional design, cognitive load, fault tolerance, testing).

## Step 1: Select task

If the user specified a task, use that. Otherwise show the backlog:

```bash
ls tasks/backlog/*.md 2>/dev/null
```

Pick the highest-priority task with lowest ID (respecting dependency order) unless user directs otherwise.

## Step 2: Move to in-progress

```bash
mv tasks/backlog/task-NNN.md tasks/in-progress/
```

## Step 3: Assess scope and ceremony level

Read the task file. Determine scope:
- **Light (1-3 files, <300 lines):** Standard workflow, skip checkpoints
- **Standard (4-8 files, 300-800 lines):** + plan mode + checkpoints every 5 files
- **Heavy (8+ files, 800+ lines, or 3+ directories):** Flag as oversized. Ask user if task should be split.

**Quick mode:** If task is Light AND has clear acceptance criteria AND touches well-understood code, proceed without intermediate confirmations. Not everything needs full ceremony.

State the ceremony level before proceeding.

## Step 4: Read context, challenge premises, discover reuse

Read selectively — don't load everything:
- Task description and acceptance criteria (full)
- Source PRD/TRD: only referenced sections
- Prior related tasks: only their summary sections (files changed, deviations, open items)

Then explore the codebase in parallel:
- Read files the task will modify; find sibling files for naming conventions
- Find test files near targets — note framework, assertion style, existing patterns
- **Search for reuse candidates:** Before writing new code, look for existing utilities, helpers, or similar implementations to reuse or extend. Check shared/common directories.
- **Align with existing norms:** Match the style, naming, and design of surrounding code. New code should feel native, not foreign.
- Flag anything that has changed since the design was written — the codebase is the truth.

### Challenge the premise

Before implementing, verify the task actually makes sense given code reality:
- Does the code the task assumes exists actually exist? Is it shaped as described?
- Is the proposed approach the simplest way to achieve the goal, given what you now see?
- Would a different approach be materially better? (If yes: state the alternative and why, ask user before diverging on approach — but proceed without asking for small tactical adjustments.)
- Is the acceptance criteria achievable as written, or does it conflict with how the system works?

If the premise is wrong, say so with evidence before implementing. A well-executed bad idea is still bad.

## Step 5: Plan (Standard+ scope)

For Standard or Heavy tasks, use plan mode to build a concrete implementation plan before writing code:

1. **Enter plan mode** — Break the task into ordered sub-steps (aim for 3-7 steps, each independently verifiable)
2. **Each step should be small:** one concern, one file change or small group of related changes
3. **Include verification for each step:** what command or test proves this step works?
4. **Identify decision points:** where might the approach need to change based on what you find?
5. **Get user approval** before proceeding to implementation

Plan structure:
```
Step 1: [what] — verify with [how]
Step 2: [what] — verify with [how]
...
```

Keep individual steps small enough that if something goes wrong, you only redo one step — not the whole task. The plan IS the todo list.

**Light tasks skip this step** — go straight to implementation.

## Step 6: Implement with discipline

Follow these rules:

### Scope guardrails
- If you touch 3+ unplanned files: STOP. Report the deviation and ask user to confirm expanded scope.
- Stick to what the task asks. No "while I'm here" cleanups.

### Do NOT (explicit negative constraints)
- Do NOT refactor code unrelated to the task
- Do NOT add error handling for impossible states
- Do NOT introduce abstractions with only one consumer
- Do NOT add features not in the acceptance criteria
- Do NOT change public interfaces unless the task requires it
- Do NOT optimize prematurely — correctness first
- Do NOT touch comments or code unrelated to your change
- Do NOT reference issue/bug numbers in code or comments — explain the *why*, not the ticket

### Execution logging (log-then-proceed)
After each meaningful change, append to the log BEFORE starting the next change:
```
### [What you did]
**Files:** `path/to/file`
[2-4 sentences: what changed, why, patterns followed]
```

### Deviation tracking
If implementation differs from the design, use an explicit marker:
```
### [What you did instead]
**Files:** `path/to/file`
**Deviation:** Design said X, did Y because Z
```
Deviations are the highest-value artifact — they feed `/ygs-sync` reconciliation and `/ygs-retro` analysis. Never skip logging them.

**If user gives feedback mid-implementation and you modify or revert something, update the log immediately.**

### Design principles
- Use interfaces + dependency injection for stateful components; stateless logic needs neither
- Test the public contract, not internals — call real methods and assert side effects/return values
- Stubs only at 3rd-party/OS boundaries (HTTP clients, clocks, filesystem, randomness)
- If you can't test without mocking internal code, the design is wrong — fix the design
- Use deep modules: small interface, rich implementation — not shallow pass-through layers
- Encapsulate internal logic cleanly; keep public surfaces minimal and unsurprising
- Think about failure modes: partial failure, concurrency, large data volumes, high request rates

### Functional design principles
- Prefer immutable data transformations over mutable state
- Make invalid states impossible with sum types / enums / sealed classes
- Use state machines (FSM) for explicit transitions — invalid sequences don't compile
- Parse, don't validate — transform at boundaries, trust internally
- Command-Query Separation — methods either change state OR return data, never both
- Errors as values (Result types) — cannot be silently ignored
- Functional core, imperative shell — pure logic separated from I/O

### Checkpoints (Standard+ scope)
Every 5 files changed, verify build/tests still pass using `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.

## Step 7: Write tests

- Write tests that would have caught the problem this task solves
- Test boundary conditions and error paths, not just happy path
- Co-locate tests with implementation when possible
- Prefer real implementations over mocks for owned code — call actual methods and assert behavior
- Tests must not be flaky: no timing-sensitive sleeps, no shared mutable state, no concurrency races
- Use deterministic time controls (fake clocks) instead of real timers where timing matters
- Keep tests simple and direct; avoid over-engineering test infrastructure
- Check for: unused variables, empty test bodies, dead code — none should ship

## Step 8: Simplify (after tests pass)

Once the initial implementation is verified, run `/simplify` to catch reuse opportunities, dead code, and complexity before moving on. Only apply when the change is non-trivial (>10 lines or control-flow changes). Re-run tests after each simplification pass to confirm no regressions. Max 2 passes.

## Step 9: Final verification — prove it works

Run the full test suite using `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.

Then **exercise the actual behavior** (not just tests):
- Run the feature/fix end-to-end if possible (dev server, CLI invocation, REPL)
- If you can't run it, explicitly state what you couldn't verify and why
- Passing tests are necessary but not sufficient — tests can be wrong

Then verify acceptance criteria systematically:
- For each acceptance criterion: trace through the code path that satisfies it
- For each new code path: confirm a test covers it
- For each error path: confirm it's tested or provably unreachable

If gaps found: write the missing test, re-run. **Max 3 QA rounds** — after 3, report status and let user decide.

## Step 10: Self-review (before requesting external review)

Before moving to done, self-check your own work:
- Re-read the acceptance criteria — does the implementation satisfy each one?
- Review the diff as a principal engineer: debug code, TODOs, hardcoded values, issue number references in comments
- Check for unintended scope expansion: did you touch files beyond what was planned?
- Verify naming consistency with surrounding code; no surprises
- Check: unused variables, empty tests, dead code, ungated debug logs
- Verify: no circular imports introduced; all new dependencies are intentional
- Confirm changes follow existing architecture — proportional to the problem, not over/under-engineered
- Check all ways this code can fail: partial failure, concurrent access, large inputs, high throughput
- If issues found: fix and re-run verification (max 2 self-review cycles, then flag concerns)

## Step 11: Move to done

Only after verification passes:

```bash
mv tasks/in-progress/task-NNN.md tasks/done/
```

## Step 12: Completion

Report **DONE** with:
- What was implemented
- Files changed (count + list)
- Tests added/passing
- Any deviations from design

If scope exceeded or issues arose: **DONE_WITH_CONCERNS** or **BLOCKED**.
Suggest: `/ygs-code-review` before committing.
