# Functional Design Principles

Canonical checklist for functional and compositional design. Referenced by implement and review skills — do not duplicate inline.

## Core principles

### Immutability by default
Prefer immutable data structures. Mutation is the primary source of unexpected state — make it explicit and rare.

### Parse, don't validate
Convert untrustworthy input into typed, validated domain objects at the boundary. Once inside the system, code works with valid types, not raw strings with runtime checks scattered everywhere.

### Make invalid states unrepresentable
Design types so the compiler rejects invalid combinations. A `Result` that can be both success and error simultaneously should be impossible to construct, not just convention.

### Errors as values
Return errors explicitly (`Result<T, E>`, `Either`, `Option`) rather than throwing exceptions for expected failure paths. Exceptions are for unexpected failures (panics, bugs), not business logic errors.

### Finite state machines for lifecycle
Model any object with a lifecycle (order, connection, task) as an explicit FSM. State transitions are functions; invalid transitions are compile errors, not runtime checks.

### Functional core / imperative shell
Push side effects (I/O, mutation, time, randomness) to the edges. The core of the logic is pure functions that are trivially testable. The shell orchestrates effects and calls the core.

### Command-Query Separation (CQS)
Functions either change state (commands) or return values (queries) — not both. Mixed functions are hard to reason about and test.

### Composition over inheritance
Prefer small, composable functions/modules over deep inheritance hierarchies. Flat and explicit beats deep and implicit.

## Review checklist

When reviewing code for functional design:

- [ ] Are domain boundaries expressed as types, not runtime checks?
- [ ] Are errors returned as values, not thrown for expected paths?
- [ ] Is mutable state minimized and isolated?
- [ ] Is each function either a command or a query, not both?
- [ ] Are FSMs explicit, or is lifecycle hidden in if/else chains?
- [ ] Is I/O at the edges, or scattered through business logic?
- [ ] Can the core logic be tested without I/O setup?
