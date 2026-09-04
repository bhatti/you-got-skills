# Subagent Dispatch

Patterns for dispatching isolated subagents with proper context. Used by skills that orchestrate parallel or sequential subagent work.

## The Isolation Principle

Each subagent gets a self-contained brief — not a dump of session history. A subagent that starts with "based on our earlier discussion" will produce lower-quality output than one given clean, specific context.

**Session history is your context, not the subagent's context.**

## Good Prompt Structure

Every subagent brief must contain:

1. **Focused scope** — one task, one domain, clear boundary ("review only the auth module", not "review the codebase")
2. **Relevant context** — file paths, relevant interfaces, specific question to answer. Not conversation history.
3. **Required output format** — exactly what to return (findings list, JSON, summary table). Ambiguous output format produces ambiguous output.
4. **What NOT to do** — explicitly state the scope boundary to prevent scope drift

Template:
```
You are a [role]. Your task is: [specific task].

Context:
- File: [path] — [what's relevant about it]
- Interface: [relevant signature or schema]
- Prior finding (if relevant): [specific prior finding only]

Task: [precise question or action]

Output: [exact format — e.g., "a ranked list of findings with file:line, severity, description"]
Do not: [explicit exclusions]
```

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Passing full session history | Subagent anchors on noise, misses signal | Write a focused brief with only relevant facts |
| "Figure out what's wrong" with no context | Produces generic advice | Give specific file paths, specific symptoms |
| Assuming shared state ("build on what the other agent found") | Subagents don't share state | Pass explicit outputs as inputs |
| Unspecified output format | Incompatible formats make integration hard | Specify exact schema or format |
| Delegating exploration ("look around and report") | Open-ended agents produce open-ended outputs | Give a specific hypothesis to confirm or deny |

## Integration Patterns

**Parallel findings → merge:** Collect all finding lists, sort by severity, deduplicate (same file:line from multiple agents = one finding, take highest severity). If two agents contradict each other on the same point, investigate directly — don't guess which is right.

**Sequential pipeline:** Output of Agent A becomes input to Agent B. Pass only the output, not the full Agent A transcript.

**Fail fast:** If any agent returns a CRITICAL finding, stop dispatching remaining agents and surface immediately. Critical findings invalidate the completion gate.

## When NOT to Parallelize

Use sequential single-agent work when:
- Tasks have sequential dependencies (B needs A's output)
- Tasks share mutable state (both write to the same file or DB)
- The task is exploratory debugging — one hypothesis at a time is more reliable than three simultaneous guesses
- Build → test → deploy stages — these are ordered by nature

Parallel dispatch is for **genuinely independent domains**, not for speed-running sequential work.
