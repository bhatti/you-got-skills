---
name: ygs-refine-architecture
description: Refine system architecture through structured questioning — challenges module depth, identifies shallow modules, tests seam boundaries. For larger changes spanning multiple components. Best with reasoning model (Opus-class).
---

# Refine Architecture

## Approach

Interview the user relentlessly about their architectural vision until reaching shared understanding. Challenge shallow designs, push for deep modules, and resolve ambiguity at system boundaries.

**Ask questions one at a time. Provide your recommended answer for each. Wait for feedback before continuing.**

If a question can be answered by exploring the codebase, explore the codebase instead of asking.

For distributed systems patterns, read `references/distributed-systems.md`.

## Architecture vocabulary

Use these terms precisely:

- **Module** — anything with an interface and an implementation (function, class, package, service)
- **Interface** — everything a caller must know: types, invariants, error modes, ordering, config
- **Depth** — leverage at the interface. **Deep** = lots of behavior behind a small interface. **Shallow** = interface nearly as complex as implementation (avoid)
- **Seam** — where an interface lives; a place behavior can be altered without editing in place
- **Adapter** — a concrete thing satisfying an interface at a seam
- **Leverage** — what callers get from depth
- **Locality** — what maintainers get from depth: change/bugs concentrated in one place

### Key principles
- **Deletion test**: imagine deleting the module. If complexity vanishes, it was a pass-through. If complexity reappears across N callers, it was earning its keep.
- **Deep over shallow**: prefer modules with small interfaces and rich implementation
- **One adapter = hypothetical seam. Two adapters = real seam.**

## Step 1: Find existing architecture docs

```bash
ls docs/architecture/*.md 2>/dev/null
ls docs/adr/*.md 2>/dev/null
```

Read existing docs. If none exist, ask the user for their architectural vision.

## Step 2: Explore codebase

Understand current system boundaries, dependencies, module depth. Note where you experience friction:
- Where does understanding one concept require bouncing between many small modules?
- Where are modules **shallow** (interface nearly as complex as implementation)?
- Where do tightly-coupled modules leak across their seams?
- Which parts are untested or hard to test through their current interface?

## Step 3: Refine through questioning

### Module boundaries
- What are the key modules? Are their interfaces small or large?
- Apply the deletion test: would deleting this module concentrate complexity or disperse it?
- Is there depth here (lots of behavior hidden) or is it pass-through?

### Seams and testability
- Where are the natural seams between modules?
- Can each module be tested through its interface with real implementations?
- If mocks are required internally, what design change would eliminate that need?

### Data flow & state
- How does data move through the system? Are transformations at clear boundaries?
- Where is state created, stored, and destroyed?
- Is data immutable as it flows? Or mutated in place (aliasing risk)?
- Can you trace a request end-to-end through the architecture?
- Are state transitions explicit (FSM) or implicit (boolean flags)?
- Can invalid states be represented? If so, how to make them impossible?

### Functional architecture
- Is there a clear functional core (pure logic) vs imperative shell (I/O)?
- Are side effects pushed to the edges?
- Can pipelines be composed from independent, stateless stages?
- Are errors values in the type system (Result types), not exceptions?

### Failure and recovery
- What happens when each module fails? Blast radius?
- Where are single points of failure?
- Can the system degrade gracefully?

### Evolution
- Which parts of this will change frequently? Are those behind stable interfaces?
- What would break if you needed to swap a component?
- Is there a migration path from current state to this architecture?

**Challenge shallow modules:** "This component has 12 methods but each just delegates. That's a pass-through, not depth. What would it look like to collapse those callers into the implementation?"

## Step 4: Write

Ensure directory exists:

```bash
mkdir -p docs/architecture
```

Write to `docs/architecture/<slug>.md`.

Read `~/.claude/skills/you-got-skills/templates/design-doc.md` for structure.
Read `~/.claude/skills/you-got-skills/templates/adr.md` for recording key decisions.

## Step 5: Completion

Report **DONE** with the file path.
Suggest: `/ygs-review-architecture` for critique.
