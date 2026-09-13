# Specialist: Maintainability

Review scope: will a future reader understand this code without asking the author?

**Reference:** `~/.claude/skills/you-got-skills/skills/shared/sloppiness-metrics.md` defines the verbosity anti-pattern catalogue, CC thresholds, and benchmark context used in the Sloppiness section below. Read it if context is not already loaded.

## Naming

- Names reveal intent: a reader should know what a function does from its name alone, without reading the body
- No abbreviations that aren't universally known (prefer `userCount` over `usrCnt`)
- No leading-underscore private methods (use language visibility modifiers)
- Boolean names answer a yes/no question: `is_valid`, `has_permission`, not `check` or `validate`
- Constants are named for their meaning, not their value

## Cognitive load

- Functions fit in a single screen (~24 lines); longer functions should be split at logical seams
- No more than 3-4 levels of nesting — deep nesting is a signal to extract
- No more than 7 things in a single code section (working memory limit)
- Guard clauses over nested conditionals: `if err != nil { return }` before the happy path

## Comments

- Comments explain WHY, not WHAT — the code explains what
- No comments describing what was removed, moved, or changed ("previously this was X")
- No issue/ticket numbers in code comments — those belong in commit messages
- Non-obvious invariants, workarounds, and subtle constraints are documented

## Dead code

- No unused imports, variables, or function parameters
- No commented-out code blocks
- No backward-compatibility shims for unreleased code
- No functions that are never called

## Duplication

- 3+ copies of the same logic is a pattern that should be extracted
- Check shared/common directories before flagging — the canonical version may already exist

## Comprehension (principle of least surprise)

- Does this code behave as a reader familiar with the codebase would expect?
- Are there any side effects hidden behind innocent-looking function calls?
- Does a function named `get*` actually mutate state?

## Sloppiness signals (AI-code aware)

These patterns appear at 2× the rate in AI-generated code vs. established repos (see `shared/sloppiness-metrics.md`). Flag only when clearly present in the diff — never infer.

**Verbosity patterns (flag any 2+ in the same diff as a combined finding):**
- Trivial delegators: a method whose entire body is a single forwarding call, no transformation or side effect added
- Wrapper-of-wrapper: a new class wrapping an existing class with no logic of its own
- Boilerplate getters/setters generated for every field, including ones never accessed externally
- Duplicated guard blocks (`if x is None`, `if not authorized`) at multiple call sites instead of centralizing in the callee
- Shallow pass-through functions that only rename or reorder arguments

**Cyclomatic complexity (CC) thresholds — count `if/else if/for/while/case/catch/&&/||` + 1:**

| CC | Risk | Severity |
|----|------|----------|
| 1–5 | Low, easy to test | No action |
| 6–10 | Medium, each branch needs a test | Monitor |
| > 10 | High, hard to test | **SHOULD** split |
| > 15 | Very high, virtually untestable | **MUST** split |

**Function mass heuristic (CC × √lines):** A 100-line function with CC=15 has mass 150 — both split separately (long AND complex) and flag as MUST.

## Severity guidance

| Severity | Example |
|----------|---------|
| MUST | Function with CC > 15 (virtually untestable) |
| SHOULD | Function >100 lines with multiple distinct concerns |
| SHOULD | Function with CC > 10 introduced in this diff |
| SHOULD | Name that obscures intent, requires reading the body to understand |
| SHOULD | 2+ verbosity anti-patterns from the same diff (see Sloppiness section) |
| SHOULD | Non-obvious invariant with no explaining comment |
| MAY | Minor naming preference, style suggestion |
| MAY | Missing comment on a moderately complex but not critical path |
