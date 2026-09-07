# Specialist: Architecture Drift

Review scope: cross-boundary imports, duplicate abstractions, large files needing splits, shallow wrappers, and conflicting changes between teams.

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

## Severity guidance

| Finding | Severity | Confidence |
|---------|----------|-----------|
| Import crossing architectural layer boundary (confirmed by reading file) | CRITICAL | HIGH |
| 3+ files with same basename in different modules | HIGH | HIGH |
| File >2000 lines (confirmed by wc -l) | HIGH | HIGH |
| Single function touched by 4+ authors in hotspot file | HIGH | MEDIUM |
| File 1000–2000 lines | MEDIUM | HIGH |
| Duplicate function names in 2 different modules | MEDIUM | HIGH |
| Shallow pass-through (wrapper delegates 1:1) | MEDIUM | MEDIUM |

## Finding format

```
#### [ARCH] <title> — <file>:<line> | Confidence: HIGH
**Evidence:** `grep -rn "from.*db" src/api/` → `src/api/handler.ts:12: import { User } from '../../db/models/user'`
**Impact:** Domain logic depends on persistence layer — any DB schema change breaks the handler directly.
**Recommendation:** Introduce a repository interface; inject it through the service layer.
```
