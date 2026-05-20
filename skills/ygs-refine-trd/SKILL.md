---
name: ygs-refine-trd
description: Refine a Technical Design Document through structured questioning — challenges architecture, tests assumptions against code, resolves design ambiguity until shared understanding. Best with reasoning model (Opus-class).
---

# Refine TRD

## Approach

Interview the user relentlessly about every aspect of their technical design until reaching shared understanding. Walk down each branch of the design tree, resolving dependencies one-by-one.

**Ask questions one at a time. Provide your recommended answer for each. Wait for feedback before continuing.**

If a question can be answered by exploring the codebase, explore the codebase instead of asking.

## Step 1: Find the TRD and source PRD

If not specified:

```bash
ls -t docs/trd/*.md 2>/dev/null | head -5
ls -t docs/prd/*.md 2>/dev/null | head -5
```

Read both the TRD (or user's rough notes) and the source PRD.

## Step 2: Read templates and explore codebase

Read `~/.claude/skills/you-got-skills/templates/trd.md` as structural reference for focused TRDs.
Read `~/.claude/skills/you-got-skills/templates/design-doc.md` for comprehensive design documents (larger scope).

Explore the codebase to understand current architecture, patterns, and constraints.

## Step 3: Refine through questioning

Walk through each dimension:

### Architecture fit
- How does this fit into the existing system? Which modules are touched?
- Are you introducing a new module? Is it deep (small interface, lots of implementation) or shallow?
- Could an existing module absorb this responsibility instead?

### Interface design
- What's the public interface of the new component?
- Can you reduce the number of methods/endpoints? Simplify parameters?
- Is the interface hiding complexity or just passing it through?

### Data model
- What new state is introduced? Where does it live?
- What happens to this state during failures? Is it recoverable?
- Schema migration path? Backwards-compatible?

### Dependencies and seams
- What external services does this call? Timeout/retry/circuit-breaker configured?
- Where are the seams (boundaries where behavior can change without editing in place)?
- Can this be tested with real implementations at seams, or are mocks required?

### Entity model
- What are the core domain entities? What are their relationships (1:1, 1:N, N:M)?
- Are DTOs clearly separated from domain entities? Is the boundary explicit?
- Could existing data structures serve this need, or is new state truly required?
- Conservative constraint: don't refactor what works — extend existing structures unless new requirements demand it.

### Requirements traceability
- Does every PRD requirement (MUST/SHOULD) map to a design component?
- Can you trace each Given/When/Then scenario to a testable code path?
- Are there design elements that don't trace back to any requirement? (scope creep)

### Design decisions
- For each key choice: what alternatives were considered? Why this one?
- Would a future reader understand "why" without asking? If not, document it.
- Are any decisions hard to reverse? Those warrant an ADR.

### Failure modes
- What happens when each component fails? Blast radius?
- Can the system partially succeed and leave inconsistent state?
- What's the rollback/migration path?

### Testability
- Can you test this through its public interface without mocking internal code?
- If you can't test without mocks, the design is wrong — what would fix it?
- Where are the correct test seams?
- Do acceptance scenarios from PRD map cleanly to integration tests?

### Norms and safeguards
- What coding standards apply? (Error handling patterns, naming, annotation usage)
- What are the non-negotiable constraints? (Performance budgets, security requirements, backward compatibility)
- What should the implementation explicitly NOT do? (Negative constraints prevent AI from improvising)

### Operations sequence
- What is the implementation order based on dependencies? (Entities before services before controllers)
- Are there circular dependencies? If so, the design needs restructuring.
- Can each step be independently verified before proceeding to the next?

### Proportionality
- Is the complexity justified for this problem size?
- Could a simpler approach work? What would you lose?
- Are you building for hypothetical future requirements?

**Cross-reference with code:** When the user states how something works, check whether the code agrees. Surface contradictions.

**Challenge over-engineering:** "You're adding three layers of abstraction for one consumer. Is that depth or ceremony?"

## Step 4: Write the refined TRD

Ensure directory exists:

```bash
mkdir -p docs/trd
```

Write (or update) at `docs/trd/YYYY-MM-DD-<slug>.md`. Link back to source PRD.

## Step 5: Completion

Report **DONE** with the file path.
Suggest: `/ygs-review-trd` for critique, `/ygs-wbs` to break into tasks.
