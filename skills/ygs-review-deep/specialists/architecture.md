# Specialist: Architecture & Design

Review scope: does the structure of this change belong in the existing design, or does it introduce technical debt?

For functional design principles, read `~/.claude/skills/you-got-skills/skills/shared/functional-design.md`.

## Module boundaries

- Does this change introduce a new import cycle?
- Does a dependency flow in the wrong direction (e.g., domain importing from infrastructure)?
- Is a new module necessary, or is this an existing module that should be extended?
- Deletion test: if this new abstraction were removed, would its complexity disperse to N callers? If yes, it is earning its keep. If not, it is a shallow wrapper.

## Abstraction level

- Is the new code at the right level of abstraction?
- Too deep: everything is an interface with a single implementation — over-engineered
- Too shallow: duplicated logic across callers that should be extracted — under-engineered
- Does this introduce a 3rd+ copy of logic that should be unified?

## Cohesion with existing design

- Does the new code follow the naming conventions, module structure, and patterns of the surrounding code?
- Is a new paradigm or framework introduced without justification? New patterns require context.
- Does the SDK/wrapper layer contain business logic that belongs in the underlying framework?

## Circular dependencies

- New circular module reference introduced — resolve with proper abstraction, not a trait workaround
- Dependency inversion missing where it would break the cycle cleanly

## Interface design (deep modules)

- Small, stable interface with a rich implementation = high leverage
- Large interface with thin implementation = shallow pass-through (anti-pattern)
- Public API exposes only what callers need; internal details stay hidden

## Data contracts

- New cross-boundary data types defined at the appropriate contract layer
- Existing entity and ID types used consistently — no ad-hoc construction or parsing where a canonical form already exists in the codebase
- No stringly-typed parameters where a typed enum or newtype would serve

## Severity guidance

| Severity | Example |
|----------|---------|
| MUST | New circular dependency, dependency direction inversion |
| MUST | Business logic in SDK/wrapper layer instead of framework |
| SHOULD | Abstraction with a single consumer, shallow pass-through module |
| SHOULD | New pattern introduced without precedent or explanation |
| MAY | Minor naming inconsistency with surrounding code |
