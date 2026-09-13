# Code Quality Principles

**Reference:** `~/.claude/skills/you-got-skills/skills/shared/sloppiness-metrics.md` — verbosity anti-patterns, CC thresholds, and benchmark numbers. The Verbosity and Complexity section below is a summary; the shared file is authoritative.

## Cognitive Load
- Methods limited to ~24 lines (80/24 rule)
- No more than 7 things in a single code section (hex flower)
- Working memory holds 4-7 pieces — code exceeding this is unmanageable
- Fractal architecture: large systems decompose into understandable chunks
- **Function names ≤ 30 characters** — longer names signal the function does too much
- **Named constants over magic values** — extract any recurring literal (number, string, flag) into a named constant or enum; exception: self-explanatory one-offs that add no meaning when named
- **Blank lines between logical sections** — use vertical whitespace to group related lines; absence of grouping forces readers to parse structure themselves

## Design Heuristics
- **Command-Query Separation** — Methods either change state OR return data, not both
- **Parse, Don't Validate** — Transform unstructured input into structured output during parsing
- **Poka-Yoke (Mistake-Proofing)** — Design interfaces that are difficult to misuse
- **Postel's Law** — Be conservative in what you send, liberal in what you accept
- **Separation of Concerns** — Things changing at same rate belong together; different rates apart
- **Private by Default** — All members start private; visibility changes are breaking design decisions requiring explicit justification
- **Layer Adjacency** — Layers only communicate with immediate neighbors; no cross-layer shortcuts; low-level mechanics (parsing, I/O, drivers) stay in their own layer

## Deep Modules
- Small interface + rich implementation = high leverage (good)
- Large interface + thin implementation = pass-through (bad)
- Deletion test: if removing a module disperses complexity to N callers, it was earning its keep
- DRY: before creating a new abstraction, verify no existing one already solves the problem — check shared/common directories and sibling modules

## Tidying Principles (from "Tidy First")
- Tidy incrementally to enable the next behavior change, not for perfection
- Tidy first when: cost(tidy) + cost(change after tidy) < cost(change without tidy)
- Guard clauses over nested conditionals
- Normalize symmetries for consistency
- New interface, old implementation (simplify without rewriting)
- Extract helpers only when they create depth (not just shorter methods)
- One pile first: consolidate scattered code before splitting

## Implementation Patterns
- **Functional Options** — Configuration as functions for validation and defaults
- **State machines with enums** — Explicit state transitions, compile-time safety
- **Memoization** — Cache expensive computations for repeated identical inputs
- **Strangler Pattern** — Side-by-side implementation before cutover for significant changes

## Testing Discipline
- Arrange-Act-Assert structure
- Red-green-refactor cycle (vertical slices, not horizontal)
- Devil's Advocate — attempt passing tests with incomplete implementation
- Parameterized tests for systematic invariant validation
- Test through public interfaces, not internals

## Verbosity and Complexity Thresholds

**Cyclomatic Complexity (CC):** count `if / else if / for / while / case / catch / && / ||` branches + 1.

| CC | Risk | Action in review |
|----|------|-----------------|
| 1–5 | Low | No action |
| 6–10 | Medium | Each branch needs a test |
| > 10 | High | Flag **SHOULD** split |
| > 15 | Very high | Flag **MUST** split |

**Verbosity anti-patterns to flag in diff** (see `shared/sloppiness-metrics.md` for full catalogue):
- **Trivial delegators** — method body = single forwarding call, no logic added
- **Wrapper-of-wrapper** — new class wraps existing class with zero added logic
- **Duplicated guards** — same `if nil/None/not authorized` repeated at 3+ call sites
- **Shallow pass-throughs** — function that only renames args to call one other function

Flag as SHOULD when 2+ of these patterns appear together in a diff.

**Benchmark context:** AI-generated code is ~2× more verbose (verbosity ratio 0.33) and ~2× higher erosion (0.68) vs. established repos (0.15 / 0.31). These patterns are the most common contributors.

## Transaction & Concurrency
- Explicit transaction boundaries — atomicity requirements drive architecture
- Optimistic concurrency for low-contention scenarios
- Connection pool > thread count + buffer (avoid starvation)
- SAGA pattern for distributed operations with compensation
- Outbox pattern to eliminate dual-write problem
