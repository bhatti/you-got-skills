---
name: ygs-sync
description: Bidirectional sync between design docs and code — keeps TRD/architecture docs accurate as implementation evolves. Use after refactoring, code review changes, or discovering better patterns during implementation.
---

# Design-Code Sync

**Principle:** Design documents must reflect actual implementation, not just planned implementation. Stale docs are worse than no docs — they actively mislead.

## Step 1: Identify what changed

Two input sources (use both when available):

**From git:**
```bash
git diff main...HEAD --stat 2>/dev/null || git diff master...HEAD --stat 2>/dev/null
```

**From execution logs (if tasks exist):**
```bash
grep -r '\*\*Deviation:\*\*' tasks/done/ tasks/in-progress/ 2>/dev/null
```

Collect all `**Deviation:** Design said X, did Y because Z` entries — these are the highest-signal input for reconciliation.

Or ask the user which components were refactored and why.

## Step 2: Find the corresponding design docs

```bash
ls -t docs/trd/*.md docs/architecture/*.md docs/adr/*.md 2>/dev/null
```

Read the relevant design documents. Identify which sections describe the changed components.

## Step 3: Compare implementation to specification

For each changed component, read the current code and compare against the design doc:

| Dimension | Check For Drift |
|-----------|----------------|
| **Entities** | Class names, fields, relationships match? New entities not in doc? |
| **Interfaces** | Method signatures, return types, error types still accurate? |
| **Architecture** | Layer responsibilities, dependencies, data flow still as designed? |
| **Data model** | Schema matches? New fields/tables not documented? |
| **Operations order** | Implementation sequence match dependency order in design? |
| **Norms** | Patterns actually used match documented standards? |
| **Constraints** | Performance budgets, security requirements still enforced? |

Categorize each discrepancy:
- **Structural** — Class hierarchy, dependencies, component boundaries changed
- **Behavioral** — Method logic, validation rules, error handling differs
- **Naming** — Classes/methods/fields renamed for clarity
- **Addition** — New components, methods, or fields not in design
- **Removal** — Components removed or deprecated

## Step 4: Present sync plan

Before modifying any docs, present proposed changes as **specific diffs** (not vague summaries):

```
## Sync Plan

### Proposed Edits

| # | Doc | Section | Current text | Proposed text | Source |
|---|-----|---------|-------------|---------------|--------|
| 1 | TRD | §3.2 | "Service uses Strategy pattern" | "Service uses match on 2-variant enum (YAGNI)" | Task 4 deviation |
| 2 | Arch | §2.1 | No mention of caching | Add: "Redis read-through cache at API boundary" | git diff |

### Drift Warnings
[If 3+ deviations cluster in the same design section, flag it:]
⚠️ 4 deviations in "Authentication" section across tasks 2, 3, 5, 7 — design assumptions in this area may no longer hold. Consider re-review.

### ADR Needed?
- [ ] Architectural decision reversed: was X, now Y because Z
```

**Ask user to confirm before proceeding.** Group related deviations to minimize edit count.

## Step 5: Apply updates

Update design documents following these rules:
- Preserve existing structure and formatting
- Match the level of detail already present (don't over-document)
- Document the "why" for significant deviations (not just the "what")
- Add ADR if an architectural decision was reversed or significantly changed

For significant deviations, add a brief rationale:
> "Design specified Strategy pattern; implementation uses simple switch because only 2 variants exist and YAGNI applies."

## Step 6: Validate consistency

After updates, verify:
- **Internal consistency** — Entities referenced in operations exist in entity section
- **Traceability** — Requirements still map to implementation components
- **No orphaned references** — Old class/method names fully replaced
- **Completeness** — All new components have corresponding documentation

## Step 7: Completion

Report **DONE** with:
- Docs updated (list)
- Changes made per doc (summary)
- ADRs created (if any)
- Remaining inconsistencies that need human judgment

Or **NEEDS_INPUT** if deviations are large enough to warrant design re-review.
Suggest: `/ygs-review-trd` or `/ygs-review-architecture` if changes were significant.

## When to Use

- After code review led to refactoring
- After discovering better patterns during implementation
- After bug fixes required logic changes
- After performance optimization changed architecture
- After adding components not in original design
- Periodically during long-running feature work (weekly sync)
