# Specialist: Maintainability

Review scope: will a future reader understand this code without asking the author?

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

## Severity guidance

| Severity | Example |
|----------|---------|
| SHOULD | Function >100 lines with multiple distinct concerns |
| SHOULD | Name that obscures intent, requires reading the body to understand |
| SHOULD | Non-obvious invariant with no explaining comment |
| MAY | Minor naming preference, style suggestion |
| MAY | Missing comment on a moderately complex but not critical path |
