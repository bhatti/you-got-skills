# Refine Scaffold

Common protocol for all refine skills. Individual skills define WHAT dimensions to question — this defines HOW to run a refinement session.

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — you own the quality of the output document.

## Questioning protocol

- Ask questions **one at a time**
- Provide your **recommended answer** for each
- **Wait for feedback** before continuing
- If a question can be answered by exploring the codebase, explore instead of asking

## Challenge patterns

- **Vague language:** "fast", "easy", "scalable" → push for specifics ("Fast means what? 100ms? 1s?")
- **Terminology conflicts:** Check against CONTEXT.md and code naming. Call out overloaded terms.
- **Code contradictions:** When user states how something works, verify against code. Surface discrepancies.
- **Concrete scenarios:** For each requirement/decision, invent a specific scenario that probes edge cases.
- **Wrong premises:** If the user's description of how the system works doesn't match what you see in code, say so with evidence. Don't refine a document built on false assumptions.
- **Simpler alternatives:** If you can see a materially simpler approach that achieves the same goal, raise it. Don't just polish what's proposed.

## Do NOT (negative constraints)

- Do NOT add requirements/features the user didn't ask for
- Do NOT assume features from similar products should be included
- Do NOT expand scope beyond what the user describes
- Do NOT write the output document until questioning reaches shared understanding

## CONTEXT.md maintenance

When a domain term is resolved during questioning, update or create `CONTEXT.md` immediately:
- Only domain-specific terms (not general programming concepts)
- One sentence definitions
- List aliases to avoid
- Create lazily if it doesn't exist

## ADR discipline

Only offer an ADR when **all three** are true:
1. Hard to reverse
2. Surprising without context
3. Real trade-off (genuine alternatives existed)

If any criterion is missing, skip the ADR.

## Directory convention detection

```bash
# Check if hierarchy exists or use flat structure
ls docs/<type>/*/*/ 2>/dev/null && echo "hierarchy" || echo "flat"
```

- **Flat:** `docs/<type>/YYYY-MM-DD-<slug>.md`
- **Hierarchy:** `docs/<type>/{product}/{project}/YYYY-MM-DD-<slug>.md`
