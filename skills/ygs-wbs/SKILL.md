---
name: ygs-wbs
description: Work Breakdown Structure — hierarchically decompose PRD/TRD into vertical-slice tasks with dependency ordering, scope assessment, and acceptance criteria. Each task cuts through all layers end-to-end.
---

# Work Breakdown Structure (WBS)

A WBS hierarchically divides a complex project into smaller, manageable components — making it easier to estimate costs, assign resources, and track progress.

## Step 1: Find source documents

```bash
find docs/prd -name "*.md" -type f 2>/dev/null | sort -r | head -5
find docs/trd -name "*.md" -type f 2>/dev/null | sort -r | head -5
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

## Step 4: Read task template, verify codebase reality, discover reuse

Read `~/.claude/skills/you-got-skills/templates/task.md` for structure.
Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — challenge the design if it doesn't match code reality.

Before decomposing, verify the PRD/TRD assumptions against the actual codebase:
```bash
# Find existing patterns, utilities, similar features
grep -rl "relevant_keyword" src/ lib/ 2>/dev/null | head -10
```

- Does the code structure match what the design assumes? (Module boundaries, existing interfaces, actual dependencies)
- Are there existing implementations that make whole tasks unnecessary?
- Does the proposed architecture conflict with how the system actually works?
- If the design says "modify X" — does X exist? Is it shaped as described?

If the PRD/TRD is based on stale or incorrect understanding of the codebase, flag it before decomposing. A precise breakdown of a wrong design just makes the wrong thing happen faster.

Note reuse candidates in task descriptions — "extend existing X" is better than "build new Y" when X already does 80% of what's needed.

## Step 5: Hierarchical decomposition

### WBS levels
1. **Deliverable** — Major feature or component (from PRD/TRD)
2. **Work package** — Independently shippable unit of work
3. **Task** — Atomic implementation step (what gets a task file)

### EARS traceability in acceptance criteria

Each task's acceptance criteria must reference the EARS pattern it satisfies:
- `WHEN [trigger] → system shall [response]` — event-driven requirement
- `WHILE [state] → system shall [behavior]` — state-driven requirement
- `IF [fault] → then system shall [response]` — unwanted behaviour (error path)
- `WHERE [feature] → system shall [behavior]` — optional feature gate
- `THE SYSTEM SHALL [behavior]` — ubiquitous invariant

This creates a direct chain: **PRD requirement → EARS pattern → acceptance criterion → test**. If an AC can't be tied to an EARS pattern, it's either missing (go back to PRD) or gold-plating (remove it).

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

### Milestone discipline

Each task is still a vertical slice (INVEST above). These rules govern how slices are **ordered** and **grouped into milestones**:

- **Every milestone ships value.** No milestone exists purely to "make progress." Each delivers something a user (internal or external) can use, adopt, or verify. If a candidate milestone's only validation is "code compiles, tests pass" — fold it into the next milestone that actually delivers.
- **Milestone validation statement:** "Who gets value, what they can now do" — not "tests pass."
- **Minimum code per milestone.** One adapter, not all four. One field, not the full set. Hardcoded first, dynamic later.
- **Order simpler slices first.** Within a milestone, prefer slices that establish foundational types/traits before slices that compose them — this reduces rework when interfaces evolve. But each slice still cuts end-to-end.
- **Balance milestones.** Avoid 1 task in M1 and 8 in M2. 7+ tasks should have 3+ milestones.

### Dependency detection
- Data model tasks before API tasks
- API tasks before UI tasks
- Migration tasks before feature tasks
- Shared utilities before consumers

### Execution mode classification (HITL vs AFK)

Every task gets classified:
- **AFK (Away From Keyboard)** — fully specified, can be completed by an agent without human judgment. Has concrete acceptance criteria, no design decisions required.
- **HITL (Human-in-the-loop)** — requires human involvement: architectural decisions, design review, external system access, manual testing, judgment calls about UX.

Preference: maximize AFK tasks. If a task is HITL only because it lacks specificity, sharpen the spec until it becomes AFK. If it's HITL because it genuinely requires human judgment, mark it explicitly and explain why.

### Dependency waves (parallel execution groups)
Group tasks into waves — independent tasks within a wave can execute in parallel, waves execute sequentially:
- **Wave 1:** Foundation (schema, shared types, config) — no dependencies
- **Wave 2:** Core logic (services, domain) — depends on Wave 1
- **Wave 3:** Integration (APIs, UI, events) — depends on Wave 2
- **Wave 4:** Polish (docs, monitoring, cleanup) — depends on Wave 3

Mark each task with its wave number and execution mode (AFK/HITL). Tasks within the same wave have no ordering constraint between them.

## Step 6: Write task files

Create files in `tasks/backlog/` following the template. Include:
- Clear title
- Source doc reference
- Priority + estimate
- Specific acceptance criteria (not "it works")
- Notes with implementation hints if relevant

## Step 7: Self-validation

Before reporting, verify the breakdown:
- **Coverage:** Every requirement in the source doc maps to a task. Every task traces back to a requirement.
- **Sizing:** Flag tasks >8 files or with multiple unrelated "and"s — suggest specific split points.
- **Dependencies:** No circular deps. No over-serialization (sequential tasks that could be parallel). Flag bottleneck tasks (3+ dependents).
- **Acceptance criteria quality:** Each criterion is specific (names concrete things), testable (verifiable by command), and complete (covers the "Delivers" statement). Minimum 2 per task.
- **Scope contracts:** File paths verified against codebase reality (use grep/find to confirm they exist or parent dirs exist for new files).

## Step 8: Completion

Report **DONE** with:
- Number of tasks created (AFK count / HITL count)
- WBS hierarchy (deliverables → work packages → tasks)
- Summary table (ID, title, priority, estimate, wave, mode, dependencies)
- Dependency waves visualized (which tasks can run in parallel)
- Any XL tasks that need further breakdown
- Critical path highlighted
- HITL bottlenecks identified (HITL tasks blocking AFK waves)

Suggest: `/ygs-implement` to start working on tasks.
