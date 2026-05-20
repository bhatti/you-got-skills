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
- **Light (1-3 files, <300 lines):** Standard workflow
- **Standard (4-8 files, 300-800 lines):** + checkpoints every 5 files
- **Heavy (8+ files or 800+ lines):** Flag as oversized. Ask user if task should be split.

## Step 4: Read context

- Read task description, acceptance criteria, and source doc reference
- If a source PRD/TRD is linked, read it for context
- Explore relevant codebase areas to understand existing patterns

## Step 5: Implement with discipline

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

### Execution logging
After each meaningful change, note what was done and why before proceeding to the next change.

### Deviation tracking
If implementation differs from the design: document "Design said X, did Y because Z."

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

## Step 6: Write tests

- Write tests that would have caught the problem this task solves
- Test boundary conditions and error paths, not just happy path
- Co-locate tests with implementation when possible
- Prefer real implementations over mocks for owned code

## Step 7: Final verification

```bash
[ -f Makefile ] && make test
[ -f package.json ] && npm test
[ -f Cargo.toml ] && cargo test
[ -f go.mod ] && go test ./...
```

## Step 8: Move to done

Only after verification passes:

```bash
mv tasks/in-progress/task-NNN.md tasks/done/
```

## Step 9: Completion

Report **DONE** with:
- What was implemented
- Files changed (count + list)
- Tests added/passing
- Any deviations from design

If scope exceeded or issues arose: **DONE_WITH_CONCERNS** or **BLOCKED**.
Suggest: `/ygs-code-review` before committing.
