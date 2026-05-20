# Functional Design Principles

## Making Invalid States Impossible

- **Sum types (algebraic data types)** — Enumerate valid states explicitly. Each variant carries exactly its required data. Illegal state combinations cannot be represented.
- **Typestate pattern** — Encode valid state transitions into the type system. Builder methods only exist on the correct state type. Invalid sequences are compile errors.
- **Newtype pattern** — Distinct types for semantically different values (PipelineId vs RouteId). Compiler prevents mixing at zero runtime cost.
- **Pattern matching for exhaustive handling** — All cases addressed. Add a new variant → every unhandled match becomes a compile error.

## Immutability & Pure Functions

- **Immutable data transformation** — Events/data flow as immutable values. Each operation returns a new value, preventing aliasing bugs.
- **Functional core, imperative shell** — Pure computational logic separated from I/O orchestration. Domain services contain no I/O.
- **Value objects** — Immutable objects defined by attributes, not identity. Share instances freely without side effects.
- **Persistent data structures** — Share unchanged portions when modified. Multiple versions coexist safely.

## Composition & Pipelines

- **Pipes and filters** — Independent processing stages. Each receives owned input and returns new output. Composable without coupling.
- **Function composition** — Small, independently testable functions composed at runtime. Fold over function chain.
- **Monadic error propagation** — Chain operations that can fail. Errors short-circuit automatically with full context (Result type + ? operator).
- **Lazy evaluation** — Nothing computed until consumer requests it. Eliminates polling and unnecessary computation.

## State Machines (FSM)

- **Explicit state transitions** — Model states as enum/union types. Transitions are methods that consume one state and produce another.
- **Compile-time transition enforcement** — Invalid transitions don't compile. The type system IS the state machine documentation.
- **Actor model** — Each actor owns mutable state; messages are the only communication. No concurrent access, no locks, no races.

## Boundaries & Architecture

- **Hexagonal architecture (ports & adapters)** — Dependencies point inward only. Domain knows nothing of infrastructure.
- **Dependency inversion** — Depend on abstractions (traits/interfaces), not implementations. Enables testing and swapping.
- **Trait-based capabilities** — Fine-grained interfaces. Callers depend only on what they need.
- **Bounded channels for backpressure** — Fixed-size channels between components. Producers wait when buffer full.

## Error Handling

- **Errors as values** — Result<T, E> in signatures. Cannot be silently ignored.
- **Command-Query Separation** — Methods either change state OR return data. Never both.
- **Parse, don't validate** — Transform unstructured input into structured output at boundaries. Trust internally.

## Anti-Patterns (Avoid)
- Mutable shared state without ownership discipline
- Stringly-typed APIs (use enums/newtypes instead)
- God objects that accumulate unrelated state
- Exceptions for control flow (use Result types)
- Implicit state transitions (use explicit FSM)
