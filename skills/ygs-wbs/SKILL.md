---
name: ygs-wbs
description: Work Breakdown Structure — hierarchically decompose PRD/TRD into vertical-slice tasks with dependency ordering, scope assessment, and acceptance criteria. Each task cuts through all layers end-to-end.
---

# Work Breakdown Structure (WBS)

A WBS hierarchically divides a complex project into smaller, manageable components — making it easier to estimate costs, assign resources, and track progress.

## Step 1: Find source documents

```bash
ls -t docs/prd/*.md 2>/dev/null | head -5
ls -t docs/trd/*.md 2>/dev/null | head -5
```

Read the PRD and TRD. Ask user which docs to decompose if multiple exist.

## Step 2: Determine next task ID

```bash
find tasks/ -name "task-*.md" 2>/dev/null | sed 's/.*task-//' | sed 's/.md//' | sort -n | tail -1
```

Next ID = max + 1 (or 001 if none exist).

## Step 3: Ensure directory structure

```bash
mkdir -p tasks/{backlog,in-progress,done}
```

## Step 4: Read task template

Read `~/.claude/skills/you-got-skills/templates/task.md` for structure.

## Step 5: Hierarchical decomposition

### WBS levels
1. **Deliverable** — Major feature or component (from PRD/TRD)
2. **Work package** — Independently shippable unit of work
3. **Task** — Atomic implementation step (what gets a task file)

### Vertical slice principles (INVEST-compliant)
- **I**ndependent — Each task is a **thin vertical slice** through ALL layers end-to-end (not a horizontal layer)
- **N**egotiable — Tasks can be descoped or adjusted without breaking others
- **V**aluable — A completed task is demoable or verifiable on its own
- **E**stimable — Small enough to estimate with confidence (1-5 days max)
- **S**mall — Prefer many thin slices over few thick ones
- **T**estable — Each task has clear acceptance criteria (testable, not vague)
- Tasks are ordered by dependency (earlier IDs = do first)
- Reference the source document in the `source:` field

### Scope assessment per task
- **S (small):** 1-3 files, <300 lines, <half day
- **M (medium):** 4-8 files, 300-800 lines, ~1 day
- **L (large):** 8+ files or 800+ lines, 2-3 days
- **XL (oversized):** Needs further breakdown — flag immediately

### Over-scoped signals (split the task)
- "and then check...", "and verify...", "and also..." in description
- Touches 3+ unrelated modules
- Has acceptance criteria in different domains (UI + backend + migration)

Exception: Causally dependent steps ARE one task (e.g., create migration + update model + update handlers for same entity).

### Priority alignment
- P0 PRD requirements → P0 tasks
- P1 PRD requirements → P1 tasks
- Infrastructure/scaffolding needed by P0 tasks → P0 priority

### Dependency detection
- Data model tasks before API tasks
- API tasks before UI tasks
- Migration tasks before feature tasks
- Shared utilities before consumers

### Dependency waves (parallel execution groups)
Group tasks into waves — independent tasks within a wave can execute in parallel, waves execute sequentially:
- **Wave 1:** Foundation (schema, shared types, config) — no dependencies
- **Wave 2:** Core logic (services, domain) — depends on Wave 1
- **Wave 3:** Integration (APIs, UI, events) — depends on Wave 2
- **Wave 4:** Polish (docs, monitoring, cleanup) — depends on Wave 3

Mark each task with its wave number. Tasks within the same wave have no ordering constraint between them.

## Step 6: Write task files

Create files in `tasks/backlog/` following the template. Include:
- Clear title
- Source doc reference
- Priority + estimate
- Specific acceptance criteria (not "it works")
- Notes with implementation hints if relevant

## Step 7: Completion

Report **DONE** with:
- Number of tasks created
- WBS hierarchy (deliverables → work packages → tasks)
- Summary table (ID, title, priority, estimate, wave, dependencies)
- Dependency waves visualized (which tasks can run in parallel)
- Any XL tasks that need further breakdown
- Critical path highlighted

Suggest: `/ygs-implement` to start working on tasks.
