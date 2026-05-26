# Writing Agent Briefs

An agent brief is posted on an issue when it moves to `ready-for-agent`. It is the authoritative spec that an AFK agent works from. The original issue body and discussion are context — the agent brief is the contract.

## Principles

### Durability over precision

The issue may sit in `ready-for-agent` for days or weeks. The codebase will change. Write the brief so it stays useful even as files are renamed or refactored.

- **Do** describe interfaces, types, and behavioral contracts
- **Do** name specific types, function signatures, or config shapes
- **Don't** reference file paths — they go stale
- **Don't** reference line numbers
- **Don't** assume the current implementation structure will remain

### Behavioral, not procedural

Describe **what** the system should do, not **how** to implement it. The agent explores the codebase fresh and makes its own decisions.

- **Good:** "The `SkillConfig` type should accept an optional `schedule` field of type `CronExpression`"
- **Bad:** "Open src/types/skill.ts and add a schedule field on line 42"

### Complete acceptance criteria

The agent needs to know when it's done. Every brief must have concrete, testable acceptance criteria. Each criterion independently verifiable.

### Explicit scope boundaries

State what is out of scope. Prevents gold-plating and assumptions about adjacent features.

## Template

```markdown
## Agent Brief

**Category:** bug / enhancement
**Summary:** one-line description of what needs to happen

**Current behavior:**
Describe what happens now.

**Desired behavior:**
Describe what should happen after the work is complete.
Be specific about edge cases and error conditions.

**Key interfaces:**
- `TypeName` — what needs to change and why
- `functionName()` — current return vs desired return
- Config shape — new configuration options needed

**Acceptance criteria:**
- [ ] Specific, testable criterion 1
- [ ] Specific, testable criterion 2
- [ ] Specific, testable criterion 3

**Out of scope:**
- Thing that should NOT be changed
- Adjacent feature that seems related but is separate
```

## What makes a bad agent brief

- Vague descriptions ("fix the thing")
- File paths and line numbers (go stale)
- No acceptance criteria
- No scope boundaries
- Missing current vs desired behavior
- Procedural instructions instead of behavioral contracts
