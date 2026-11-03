# Review Knowledge Format

Single definition of the categories and item format used by `ygs-pr-audit`'s review-learnings
pass, the `review_learnings.json` output, and the `review-knowledge` skill that the pipeline
writes into the audited repo (`.claude/skills/review-knowledge/`). Defined once here; the
specialist and the pipeline refer to it instead of repeating it.

## Categories

| Category | What belongs here |
|----------|-------------------|
| `architecture` | module boundaries, layering, dependency direction, abstractions, design fit |
| `spec` | requirements, acceptance criteria, scope, missing/unclear intent |
| `correctness` | logic errors, edge cases, error handling, concurrency, data integrity |
| `security` | authn/authz, tenant isolation, injection, secrets, input validation |
| `sre` | timeouts, retries, observability, rollout/rollback, failure modes, on-call impact |
| `testing` | coverage, flakiness, test design, what to assert |
| `api-compat` | public contracts, backward compatibility, versioning, migrations |
| `performance` | complexity, allocations, N+1, scale limits |
| `style` | naming, idioms, readability, conventions |
| `process` | PR size, descriptions, review practice (use sparingly) |

## Checklist item

```
- [ ] <rule, imperative, generalized> — Why: <the reason, one sentence> — Look for: <code pattern that triggers it> (tribal)
```

- **Rule** is generic (no PR numbers, names or incident narrative). **Why** carries the reviewer's
  reasoning: it is what lets a junior apply the rule to a case the rule does not literally cover.
- `(tribal)` marks rules resting on history or system knowledge that is not written down elsewhere
  (past incidents, undocumented invariants, "we tried that and it broke X").
- One line per item. Files stay short: one file per category, merge-only (never rewrite existing items).

## What makes a learning worth keeping

Keep: a reviewer explained a reason, pointed at a recurring pattern, or encoded system knowledge.
Drop: taste without reasoning, one-off typos, approvals ("LGTM"), anything that restates a linter rule.
Evidence bar: 2+ PRs, or a requested senior reviewer, or flagged tribal. Never invent reasoning that is
not in the comments.
