---
name: ygs-implement
description: Implement a task from the backlog with execution discipline — scope guardrails, checkpoints, deviation logging.
---

# Implement

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
- **Heavy (8+ files or 800+ lines):** Flag as oversized. Ask user if task should be split.

**Quick mode:** If task is Light AND has clear acceptance criteria AND touches well-understood code, proceed without intermediate confirmations. Not everything needs full ceremony.

## Step 4: Read context and discover reuse candidates

- Read task description, acceptance criteria, and source doc reference
- If a source PRD/TRD is linked, read it for context
- Explore relevant codebase areas to understand existing patterns
- **Search for reuse candidates:** Before writing new code, explicitly look for existing utilities, helpers, patterns, or similar implementations that can be reused or extended. Grep for related function names, check shared/common directories, review recent similar changes.

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

### Strongly-typed language principles
- Use interfaces + dependency injection for stateful components
- Prefer static methods for stateless logic
- Test the public contract, not internals
- Stubs only at 3rd-party/OS boundaries (HTTP clients, clocks, filesystem, randomness)
- If you can't test without mocking internal code, the design is wrong — fix the design

### Functional design principles
- Prefer immutable data transformations over mutable state
- Make invalid states impossible with sum types / enums / sealed classes
- Use state machines (FSM) for explicit transitions — invalid sequences don't compile
- Parse, don't validate — transform at boundaries, trust internally
- Command-Query Separation — methods either change state OR return data, never both
- Errors as values (Result types) — cannot be silently ignored
- Functional core, imperative shell — pure logic separated from I/O

### Checkpoints (Standard+ scope)
Every 5 files changed, verify build/tests still pass:

```bash
[ -f Makefile ] && make build && make test
[ -f Cargo.toml ] && cargo build && cargo test
[ -f go.mod ] && go build ./... && go test ./...
```

## Step 7: Write tests

- Write tests that would have caught the problem this task solves
- Test boundary conditions and error paths, not just happy path
- Co-locate tests with implementation when possible
- Prefer real implementations over mocks for owned code

## Step 8: Final verification

```bash
[ -f Makefile ] && make test
[ -f package.json ] && npm test
[ -f Cargo.toml ] && cargo test
[ -f go.mod ] && go test ./...
```

## Step 9: QA verification (bounded)

After tests pass, verify acceptance criteria systematically:
- For each acceptance criterion: trace through the code path that satisfies it
- For each new code path: confirm a test covers it
- For each error path: confirm it's tested or provably unreachable

If gaps found: write the missing test, re-run verification. **Max 3 QA rounds** — if still failing after 3, report status and let user decide.

## Step 10: Self-review (before requesting external review)

Before moving to done, self-check your own work:
- Re-read the acceptance criteria — does the implementation satisfy each one?
- Review the diff as if you were a reviewer: any debug code, TODOs, hardcoded values that shouldn't ship?
- Check for unintended scope expansion: did you touch files beyond what was planned?
- Verify naming consistency with surrounding code
- If issues found: fix and re-run verification (bounded — max 2 self-review cycles, then flag concerns)

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
