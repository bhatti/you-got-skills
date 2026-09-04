---
name: ygs-parallel
description: Dispatch independent parallel subagents for 2+ tasks that have no shared state or sequential dependencies. Use when facing multiple independent investigations, reviews, or implementations that can be worked on concurrently.
argument-hint: "[describe the independent tasks]"
---

# Parallel

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.

Parallel dispatch multiplies throughput only when tasks are genuinely independent. Running dependent tasks in parallel produces race conditions in results — not speed.

## Independence Check

All three must be true before parallelizing:

1. **No shared mutable state** — Reading the same files is fine. Both writing to the same file is not. Both querying the same DB read-replica is fine. Both migrating the same table is not.
2. **No sequential dependency** — Task B does not need Task A's output as input.
3. **Fully specifiable in isolation** — You can write a complete brief for each task without referencing what the other tasks will find.

If you can't describe Task B without saying "and then based on what Task A found" — they are not independent.

## Step 1: Name the independent domains

List each independent task with a one-sentence description. If naming one requires referencing another, they are not independent — use sequential execution instead.

Example (independent):
- "Review the auth module for OWASP injection vulnerabilities"
- "Review the payment module for OWASP injection vulnerabilities"

Example (not independent):
- "Find the root cause of the login failure"
- "Fix whatever Task A identifies as the root cause"

## Step 2: Write isolated briefs

For each task, write a self-contained brief. Use `~/.claude/skills/you-got-skills/skills/shared/subagent-dispatch.md` for the brief structure.

Each brief must include:
- Specific file paths or domains to examine
- The exact question to answer or action to take
- The required output format
- What NOT to do (explicit scope boundary)

## Step 3: Dispatch in a single response

Send all agent dispatches in one message as parallel tool calls. Do not dispatch Agent 2 after waiting for Agent 1 — dispatch both at once.

## Step 4: Integrate results

Once all agents report back:
- **Findings from multiple review agents:** sort by severity, deduplicate findings at the same location (keep highest severity), reconcile any contradictions by checking the code directly
- **CRITICAL findings:** surface immediately and halt further work until addressed
- **Conflicting conclusions:** investigate directly rather than arbitrating by majority

## When NOT to Parallelize

| Scenario | Use instead |
|----------|------------|
| Multiple failures in the same component | `ygs-investigate` — likely one root cause |
| Task B depends on Task A's output | Sequential execution |
| Both tasks mutate the same files or DB | Sequential execution |
| Exploratory debugging (unknown root cause) | Single-thread: one hypothesis at a time |
| Build → test → deploy pipeline stages | Sequential — ordered by nature |
| One agent reads context, another acts on it | Sequential — the reading must complete first |

## Common Anti-Patterns

| Anti-pattern | Problem |
|-------------|---------|
| "Explore the codebase and summarize each area in parallel" | Open-ended exploration with overlapping domains produces overlapping summaries with no clear integration path |
| "Both agents fix the same bug simultaneously" | Race condition — second agent overwrites first's fix |
| "Let agents discover their own task boundaries" | Agents expand scope unpredictably; overlapping scope = duplicated work or conflicts |
| Dispatching 10+ agents on a first pass | Start with 2-3 to validate independence, then scale |

---

References:
- `~/.claude/skills/you-got-skills/skills/shared/subagent-dispatch.md`
- `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`
