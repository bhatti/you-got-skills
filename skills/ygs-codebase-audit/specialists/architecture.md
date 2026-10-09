# Specialist: Architecture Drift

Review scope: cross-boundary imports, cyclic dependencies, duplicate abstractions, large files needing splits, shallow wrappers, and conflicting changes between teams.

**Reference:** `~/.claude/skills/you-got-skills/skills/shared/quality-checklist.md` sections 3 (Cyclic Dependencies) and 4 (Modular Boundaries) — canonical detection commands and severity rules.

**Reference:** `~/.claude/skills/you-got-skills/skills/shared/design-principles.md` — SOLID checklist (Section 1), package cohesion rules (Section 2), stability metrics I/A/D (Section 3), hotspot coupling analysis (Section 4). Apply in Steps 5–7 below.

## Step 1: Cross-boundary imports

Identify the project's top-level architecture layers (e.g., `api/`, `service/`, `domain/`, `infra/`, `db/`, `model/`, `controller/`).

```bash
# Identify top-level module directories
find . -maxdepth 2 -name "*.py" -o -name "*.ts" -o -name "*.go" 2>/dev/null \
  | sed 's|^\./||' | cut -d/ -f1 | sort -u | head -20

# Find imports flowing in wrong direction (e.g., domain importing from infra)
# Adapt pattern to actual directory names found above
grep -rn "from.*infra\|import.*infrastructure\|from.*db\|require.*database" \
  --include="*.py" --include="*.ts" --include="*.go" --include="*.js" \
  src/domain/ lib/domain/ domain/ 2>/dev/null | grep -v test | head -20

# General cross-boundary: controllers/handlers importing from models/entities directly
grep -rn "from.*model\|from.*entity\|import.*schema" \
  --include="*.py" --include="*.ts" \
  src/api/ src/controller/ src/handler/ app/api/ 2>/dev/null | grep -v test | head -20
```

For each hit: read 5 lines of context around the import. Only flag if the import genuinely crosses an architectural boundary (not just a shared types import).

```bash
# Show the exact import line + surrounding context before flagging
sed -n '$((LINE-2)),$((LINE+2))p' <file> 2>/dev/null
```

## Step 2: Duplicate utilities / helpers

```bash
# Same-named utility files across modules
find . \( -name "utils.*" -o -name "helpers.*" -o -name "common.*" \
         -o -name "util.*" -o -name "shared.*" -o -name "misc.*" \) \
  ! -path "*/.git/*" ! -path "*/node_modules/*" ! -path "*/vendor/*" \
  ! -path "*/dist/*" ! -path "*/build/*" 2>/dev/null \
  | xargs -I{} sh -c 'echo "$(basename {})|{}"' \
  | sort | awk -F'|' 'seen[$1]++{print $2; if(!printed[$1]){print prev[$1]; printed[$1]=1}} {prev[$1]=$2}'

# Duplicate exported function/class names across modules
grep -rh "^export function \|^export class \|^export const \|^def \|^func \b" \
  --include="*.py" --include="*.ts" --include="*.go" --include="*.js" \
  ! -path "*/node_modules/*" ! -path "*/vendor/*" 2>/dev/null \
  | sed 's/[^a-zA-Z_].*$//' | sort | uniq -d | head -20
```

For each duplicate found: list ALL file paths containing it. Only flag if the files are in different modules (different top-level directory).

## Step 3: Large files needing module splits

```bash
find . \( -name "*.py" -o -name "*.ts" -o -name "*.go" -o -name "*.js" \) \
  ! -path "*/.git/*" ! -path "*/node_modules/*" ! -path "*/vendor/*" \
  ! -path "*/dist/*" ! -path "*/build/*" ! -name "*.min.*" \
  -exec wc -l {} + 2>/dev/null | sort -rn | head -15
```

For any file >1000 lines: read its top-level structure (class/function list) to understand if it can be split.

```bash
grep -n "^class \|^def \|^func \|^function \|^export " <large_file> 2>/dev/null | head -20
```

## Step 4: Conflicting changes between teams/authors

For the top-10 hotspot files: check if the same function/class was recently edited by multiple different authors (signals ownership conflict or missing coordination).

```bash
# Show recent authors for a hotspot file's most-changed sections
git log --pretty=format:"%an" -30 -- <hotspot_file> 2>/dev/null | sort | uniq -c | sort -rn | head -5

# Find functions touched by 3+ different authors in last 200 commits
git log --pretty=format:"%H %an" -200 -- <file> 2>/dev/null | while read hash author; do
  git show "$hash:$file" 2>/dev/null | grep -n "^def \|^func \|^function " | head -5
done 2>/dev/null | sort | uniq -c | sort -rn | head -10
```

## Step 5: SOLID violations (codebase-wide scan)

Read `shared/design-principles.md` Section 1. Apply to the top-10 hotspot files and any file >500 lines:

```bash
# SRP signals: god objects (class with 10+ public methods across unrelated concerns)
grep -rn "^class \|^def \|^func " --include="*.py" --include="*.go" --include="*.ts" \
  2>/dev/null | awk -F: '{print $1}' | sort | uniq -c | sort -rn | head -20
# Then read the top-offending files to confirm multiple unrelated public methods

# DIP signals: concrete imports in domain/service layers
grep -rn "from.*infra\|from.*db\|from.*storage\|from.*adapter\|import.*Repository\b" \
  --include="*.py" --include="*.go" \
  $(find . -type d \( -name "domain" -o -name "core" -o -name "service" -o -name "usecase" \) 2>/dev/null | head -5) \
  2>/dev/null | grep -v test | head -10

# ISP signals: large interfaces
grep -n "def \|func " --include="*.py" --include="*.go" \
  $(find . -name "*.py" -o -name "*.go" 2>/dev/null | xargs grep -l "interface\|Protocol\|ABC" 2>/dev/null | head -10) \
  2>/dev/null | awk -F: 'seen[$1]++{} END{for(f in seen) if(seen[f]>8) print seen[f], f}' | sort -rn | head -5
```

For each confirmed violation, add an `[ARCH]` finding. Report total counts for the metrics dashboard.

## Step 6: Module stability analysis

For the top-20 non-test, non-adapter modules (by file count or import frequency), estimate I = Ce/(Ca+Ce):

```bash
# For each significant module directory, estimate Ce and Ca
for MODULE_DIR in $(find . -maxdepth 3 -type d ! -path "*test*" ! -path "*node_modules*" \
  ! -path "*vendor*" ! -path "*.git*" 2>/dev/null | head -20); do
  MODULE=$(basename "$MODULE_DIR")
  CE=$(grep -rn "^from\|^import" "${MODULE_DIR}/" --include="*.py" 2>/dev/null \
    | awk '{print $2}' | cut -d'.' -f1 | sort -u | grep -v "__\|test\|${MODULE}" | wc -l)
  CA=$(grep -rn "from ${MODULE}\b\|import ${MODULE}\b" \
    --include="*.py" --include="*.go" . 2>/dev/null \
    | grep -v "${MODULE_DIR}/" | awk -F: '{print $1}' | xargs -I{} dirname {} 2>/dev/null | sort -u | wc -l)
  TOTAL=$((CE + CA))
  if [ "$TOTAL" -gt 0 ]; then
    I=$(echo "scale=2; $CE / $TOTAL" | bc 2>/dev/null || echo "?")
    echo "I=${I}  Ce=${CE}  Ca=${CA}  ${MODULE}"
  fi
done | sort -rn | head -15
```

Flag:
- I > 0.7 for a shared/core module → SHOULD (high outbound coupling, fragile)
- I < 0.3 + no abstract interfaces found → SHOULD (SAP violation: stable but concrete — zone of pain)
- Any module where SDP is violated (depends on a module with higher I than itself) → flag informational

Report mean I across non-leaf modules for the `avg_instability_score` dashboard row.

## Step 7: Design Quality Metrics table

After Steps 1–6, emit the canonical 8-row table from
`~/.claude/skills/you-got-skills/skills/shared/design-metrics-table.md`.
Use counts from this specialist pass; write `N/A` for readability (scored in maintainability pass only).

```markdown
## Design Quality Metrics

| Principle | Score | Benchmark | Signal | Notes |
|-----------|-------|-----------|--------|-------|
| SRP violations | N | 0 ideal | 🟢/🟡/🔴 | top offending file |
| DIP violations | N | 0 ideal | 🟢/🟡/🔴 | concrete dep location |
| Cyclic deps (ADP) | N | 0 | 🟢/🔴 | cycle pair or "none detected" |
| Layer violations | N | 0 ideal | 🟢/🟡/🔴 | wrong-direction import or "none" |
| Avg instability (I) | 0.x | < 0.5 (core) | 🟢/🟡/🔴 | mean across non-leaf modules |
| Hotspot coupling | N files | 0 ideal | 🟢/🟡/🔴 | top offending file |
| Readability score | N/A | > 7 | — | scored in maintainability pass |
| ISP / fat interface | N | 0 ideal | 🟢/🟡/🔴 | interface name or "none found" |
```

## Severity guidance

| Finding | Severity | Confidence |
|---------|----------|-----------|
| Import crossing architectural layer boundary (confirmed by reading file) | CRITICAL | HIGH |
| ADP cycle (circular dependency between modules) | CRITICAL | HIGH |
| DIP violation: domain/service layer imports concrete infra type | MUST | HIGH |
| 3+ files with same basename in different modules | HIGH | HIGH |
| File >2000 lines (confirmed by wc -l) | HIGH | HIGH |
| Single function touched by 4+ authors in hotspot file | HIGH | MEDIUM |
| SRP violation: confirmed god class with 3+ unrelated concerns | HIGH | MEDIUM |
| File 1000–2000 lines | MEDIUM | HIGH |
| Duplicate function names in 2 different modules | MEDIUM | HIGH |
| I > 0.7 for a core/shared module | MEDIUM | MEDIUM |
| SAP violation: I < 0.3 + fully concrete (no interfaces) | MEDIUM | MEDIUM |
| ISP violation: interface >8 methods, callers use <3 | MEDIUM | MEDIUM |
| Shallow pass-through (wrapper delegates 1:1) | MEDIUM | MEDIUM |

## Finding format

```
#### [ARCH] <title> — <file>:<line> | Confidence: HIGH
**Evidence:** `grep -rn "from.*db" src/api/` → `src/api/handler.ts:12: import { User } from '../../db/models/user'`
**Impact:** Domain logic depends on persistence layer — any DB schema change breaks the handler directly.
**Recommendation:** Introduce a repository interface; inject it through the service layer.
```
