# Out-of-Scope Knowledge Base

The `.out-of-scope/` directory stores persistent records of rejected feature requests.

## Purpose

1. **Institutional memory** — why a feature was rejected, so reasoning isn't lost when the issue is closed
2. **Deduplication** — when a new issue matches a prior rejection, surface the decision instead of re-litigating

## Directory structure

```
.out-of-scope/
├── dark-mode.md
├── plugin-system.md
└── graphql-api.md
```

One file per **concept**, not per issue. Multiple issues requesting the same thing are grouped.

## File format

```markdown
# Concept Name

One-line summary of what's rejected and why.

## Why this is out of scope

Substantive explanation referencing:
- Project scope or philosophy
- Technical constraints
- Strategic decisions

## Prior requests

- #42 — "Original request title"
- #87 — "Similar request title"
```

### Naming

Short, descriptive kebab-case: `dark-mode.md`, `plugin-system.md`. Recognizable from the filename alone.

### Writing the reason

Not "we don't want this" but **why**. Good reasons reference project scope, technical constraints, or strategic decisions. Avoid temporary circumstances ("we're too busy") — those are deferrals, not rejections.

## When to check

During triage (Step 1: Gather context), read all `.out-of-scope/*.md` files. Match by concept similarity, not keyword — "night theme" matches `dark-mode.md`.

If there's a match, surface it: "This resembles `.out-of-scope/dark-mode.md` — rejected because [reason]. Still applies?"

Maintainer may:
- **Confirm** — add new issue to "Prior requests", close
- **Reconsider** — delete/update the out-of-scope file, proceed with normal triage
- **Disagree** — issues are related but distinct, proceed normally

## When to write

Only when an **enhancement** (not a bug) is rejected as `wontfix`:
1. Check if matching `.out-of-scope/` file exists
2. If yes: append to "Prior requests"
3. If no: create new file
4. Comment on issue explaining decision and linking the file
5. Close with `wontfix`
