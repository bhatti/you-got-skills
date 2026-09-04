---
name: ygs-write-skill
description: Use when creating a new YGS skill, improving an existing skill's content or discoverability, or deciding whether to extract content to shared/. Prevents skill duplication and ensures new skills are DRY, modular, and registered correctly.
argument-hint: "[skill name or describe the capability to add]"
---

# Write Skill

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`.

## Step 1: DRY check before writing anything

Before creating anything new:

1. **Does an existing skill already cover this?** Check `README.md` skill table. A skill that overlaps with an existing one splits attention and confuses the agent about which to invoke.
2. **Is this an enhancement to an existing skill?** Edit the existing `SKILL.md` rather than creating a new skill.
3. **Is this a shared concern used by 2+ skills?** Extract to `skills/shared/<name>.md` rather than putting it in one skill's content. The shared file extraction rule: if 2+ skills would reference the same concept, it belongs in `skills/shared/`.

## Step 2: File structure

```
skills/ygs-<name>/
└── SKILL.md              # Required — the skill itself
└── references/           # Optional — detailed checklists or reference material
    └── <topic>.md        # Referenced by SKILL.md via absolute path

skills/shared/
└── <shared-topic>.md     # Only when 2+ skills reference the same content
```

## Step 3: Frontmatter

```yaml
---
name: ygs-<name>
description: <triggering conditions — see SDO below>
argument-hint: "[what the user would pass as an argument]"
---
```

The `name` must match the directory name exactly.

## Step 4: Skill Discovery Optimization (SDO)

**The `description` field is a trigger condition, not a workflow summary.**

The agent reads the description to decide whether to invoke the skill. It is NOT documentation. If the description summarizes the workflow instead of stating when to invoke it, agents either over-invoke (every code change triggers "code review") or under-invoke (they follow the description instead of reading the full skill).

**Bad (workflow summary):**
> "Multi-pass code review — runs critical pass, informational pass, tests discipline, fix-first pattern"

**Good (triggering conditions):**
> "Diff-based code review — use when a task is done, before committing, or when asked to review code. Best with reasoning model."

**Test:** Can an agent read the description and know exactly *when* to invoke this skill without reading the full SKILL.md? If no, rewrite the description.

## Step 5: Content patterns

Choose the pattern that fits the skill type:

**Discipline skill** (enforces behavior that agents skip under pressure):
- Iron law at the top — one absolute rule
- Steps are prescriptive "must" language
- Rationalization table: the 5-8 most common excuses + evidence-based rebuttals
- Red flags list: observable symptoms of "you're doing it wrong"

**Technique skill** (how-to for a specific task):
- Steps are instructional, not prescriptive
- Code examples where useful
- "When NOT to use" section to prevent over-application

**Reference skill** (lookup table, checklist, canon):
- Organized by lookup key (language, category, severity)
- No procedural steps
- Stable — changes here affect every skill that references it

## Step 6: Shared file references

Reference shared files with their absolute installed path:

```
~/.claude/skills/you-got-skills/skills/shared/<file>.md
```

Never use relative paths — skills are invoked from the user's project directory, not from this repo.

## Step 7: Register the skill

After creating a new skill directory:

```bash
./setup install
```

This creates the symlink at `~/.claude/skills/ygs-<name>` that Claude Code uses for discovery.

## Step 8: Verify

Invoke the skill in a Claude Code session. Confirm:
- The description triggers it at the right moment (not too broadly, not too narrowly)
- The content guides the correct behavior
- For discipline skills: test against a pressure scenario where an agent would be tempted to skip it

If the skill requires too much effort to invoke correctly, the description needs to be tighter. If the skill triggers when it shouldn't, the description is too broad.

---

References:
- `~/.claude/skills/you-got-skills/conventions.md`
- `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`
