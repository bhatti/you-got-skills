# Implementation Principles

## Functional Design (Make Invalid States Impossible)
- **Sum types / enums** — Enumerate valid states explicitly. Each variant carries only its required data.
- **Typestate pattern** — Encode state transitions in the type system. Invalid sequences don't compile.
- **Newtype pattern** — Distinct types for semantically different values (UserId vs OrderId). Zero-cost compile-time safety.
- **Immutable data flow** — Each operation returns a new value. No aliasing bugs.
- **Functional core, imperative shell** — Pure domain logic with no I/O; orchestration at boundaries.
- **Errors as values** — Result<T, E> types. Cannot be silently ignored. Propagate with context.
- **Pattern matching** — Exhaustive handling of all variants. New variant = compile error on unhandled cases.
- **Bounded channels** — Backpressure between components. Producers wait when buffer full.

## Cognitive Load Management
- Keep methods under 24 lines, lines under 80 chars
- No more than 7 concurrent concepts in a section
- Fractal decomposition: each level hides details while allowing drill-down
- Group code operating on same data structures (cohesion)

## Before Writing Code
- Tidy first when it makes the next change cheaper
- Guard clauses over nested conditionals
- Normalize symmetries across the codebase
- Extract explaining variables for complex expressions
- Replace magic numbers with named constants

## Writing Code
- **Command-Query Separation** — Mutations and queries are separate methods
- **Parse, Don't Validate** — Transform at boundaries, trust internally
- **Explicit Parameters** — Pass individual values, not data maps
- **State Machines** — Model distinct states with enums/algebraic types
- **Functional Options** — Configuration with validation and defaults
- **Deep modules** — Hide complexity behind small interfaces

## Concurrency & Transactions
- Define transaction boundaries explicitly before coding
- Use outbox pattern for DB + event publishing (never dual-write)
- Optimistic concurrency for low-contention, pessimistic for financial ops
- Size connection pools correctly (> thread count + buffer)
- SAGA with compensation for distributed operations

## Fault Tolerance
- Circuit breakers on external calls
- Retry with jitter (exponential backoff + randomization)
- Graceful degradation when dependencies fail
- Idempotency tokens on all create operations
- Graceful shutdown (drain in-flight on SIGTERM)
- No hard startup dependencies

## Refactoring Safely
- **Strangler Pattern** — Build alongside, cut over, delete old
- **Humble Object** — Extract logic from hard-to-test objects
- Small commits, continuous integration
- Tidy in separate commits from behavior changes
- Feature flags for incomplete functionality

## Testing During Implementation
- Write test for the specific behavior before implementing
- Test through public interface (not internals)
- Arrange-Act-Assert pattern
- Vertical slices: one test → one implementation → repeat
- Stubs only at 3rd-party/OS boundaries
