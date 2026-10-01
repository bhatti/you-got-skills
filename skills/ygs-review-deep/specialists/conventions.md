# Specialist: Conventions

Review scope: naming consistency, error-handling patterns, async style, and AI failure modes in the diff. This dimension is always ACTIVE — conventions apply to every changed file.

Tag all findings `[CONV]`. Max 3 findings from this specialist (conventions are advisory unless they introduce actual bugs or contradict the repo's stated rules).

---

## Step 1: Load repo-local rules first

Project-specific conventions take precedence over all general rules below.

```bash
for f in CLAUDE.md AGENTS.md CONTRIBUTING.md .claude/AGENTS.md; do
  [ -f "$f" ] && echo "=== $f ===" && cat "$f"
done
```

Extract the conventions section (naming, error handling, async patterns, import rules). These override the general checks below on any conflict — flag deviations from the project rules as `[CONV] MUST`.

---

## Step 2: AI failure modes

Load and apply `~/.claude/skills/you-got-skills/skills/shared/ai-failure-modes.md`.

Check the diff for all 8 patterns. Each confirmed match is a `[CONV]` finding with the pattern name as the title. AI failure modes frequently slip through logic and testing specialists because they look structurally correct.

---

## Step 3: Naming consistency

Compare names in the diff against naming patterns in unchanged surrounding code:

- **Boolean variables/properties:** does the project use `isX`/`hasX`/`canX` prefixes? Flag deviations.
- **Casing:** is the diff consistent with the project's casing convention (camelCase, snake_case, PascalCase, kebab-case per context)?
- **File naming:** do new files follow the project's file-naming pattern?
- **Function names match behavior:** a function named `getX` that also mutates state, or `validateX` that also transforms — flag the mismatch between name and behavior.
- **Abbreviations:** does the diff introduce new abbreviations inconsistent with the project vocabulary?

Only flag naming that creates genuine confusion or breaks established project patterns. Do not flag personal preference differences that aren't contradicting a stated rule.

---

## Step 4: Error-handling pattern consistency

Identify the project's error-handling style from the unchanged surrounding code:
- **Errors-as-values** (Result/Either types, error return values)
- **Exception-based** (thrown exceptions, try/catch)
- **Mixed** (project uses both — is the diff consistent with which style applies where?)

Flag if the diff:
- Introduces an inconsistent error-handling style in a module that uses one approach uniformly
- Swallows an error that the surrounding code would propagate (see `ai-failure-modes.md` pattern 4)
- Wraps errors without preserving cause chain (e.g., `throw new Error("failed")` vs `throw new Error("failed", { cause: e })`)
- Uses a different log level than surrounding code for equivalent error severity

---

## Step 5: Async pattern consistency

Identify the project's async style from surrounding code (callbacks, promises/futures, async/await, coroutines, reactive streams).

Flag if the diff:
- Mixes async patterns in a module/file that uses one style uniformly (e.g., `await` calls inside a promise chain)
- Uses sequential async where the surrounding code uses concurrent dispatch (see `ai-failure-modes.md` pattern 3)
- Introduces a new async pattern not present elsewhere in the module without justification

---

## Step 6: Module structure and imports

- **Circular dependency:** does the diff introduce a new circular import? (Trace the import chain; flag as `[CONV] MUST` if a cycle is introduced)
- **Import style consistency:** does the diff follow the project's import ordering/grouping convention?
- **Barrel export consistency:** if the project uses barrel exports (`index.ts`, `__init__.py`), are new public symbols exported consistently?
- **Dead exports:** new symbols exported but with no callers anywhere in the codebase (check for usage before flagging — may be a new public API)

---

## Step 7: Dead conventions

- **Stale comments:** comments that describe behavior the code no longer implements, or reference old module/function names
- **Empty catch blocks:** `catch (e) {}` with no logging, no re-throw, no intentional suppression
- **Commented-out code:** blocks of code in comments with no explanation; in a PR these should be removed, not shipped

---

## Finding format

```
[CONV] <Title> — <file>:<line>
Confidence: HIGH | MEDIUM | LOW
Pattern: <which rule triggered this>
Issue: <what the diff does vs. what the convention expects>
Fix: <concrete suggested change>
```
