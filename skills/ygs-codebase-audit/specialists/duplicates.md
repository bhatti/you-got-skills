# Specialist: Duplicate Abstractions

Review scope: same logic implemented in multiple places, same-named files across modules, duplicate exported symbols that should be unified into a shared library.

## Step 1: Same-named utility/helper files

```bash
# Find files with the same basename in different directories
find . \( -name "utils.*" -o -name "helpers.*" -o -name "common.*" \
         -o -name "util.*" -o -name "shared.*" -o -name "constants.*" \
         -o -name "types.*" -o -name "errors.*" -o -name "logger.*" \) \
  ! -path "*/.git/*" ! -path "*/node_modules/*" ! -path "*/vendor/*" \
  ! -path "*/dist/*" ! -path "*/build/*" ! -path "*/__pycache__/*" \
  2>/dev/null | sed 's|.*/||' | sort | uniq -dc | sort -rn | head -10
```

For each basename appearing 3+ times: list all paths and show line counts.

```bash
# List all matching files with line counts
find . -name "<duplicate_basename>.*" \
  ! -path "*/.git/*" ! -path "*/node_modules/*" ! -path "*/vendor/*" \
  2>/dev/null | xargs wc -l 2>/dev/null | sort -rn
```

For the two largest: briefly compare their exported symbols to confirm they overlap (not just sharing a name).

```bash
# Compare exported symbols between two same-named files
grep -n "^export\|^def \|^func \|^public " <file1> 2>/dev/null | head -10
grep -n "^export\|^def \|^func \|^public " <file2> 2>/dev/null | head -10
```

## Step 2: Duplicate exported function/class names

```bash
# Functions/classes with the same name exported from different modules
grep -rh "^export function \|^export class \|^export const \|^export default function " \
  --include="*.ts" --include="*.js" \
  ! -path "*/node_modules/*" ! -path "*/dist/*" 2>/dev/null \
  | sed 's/export \(function\|class\|const\|default function\) //; s/[(<{].*//' \
  | sort | uniq -d | head -20

# Python
grep -rh "^def \|^class " \
  --include="*.py" \
  ! -path "*/node_modules/*" ! -path "*/__pycache__/*" 2>/dev/null \
  | sed 's/[(:].*//' | sort | uniq -d | head -20

# Go
grep -rh "^func [A-Z]" --include="*.go" 2>/dev/null \
  | sed 's/func //; s/[( ].*//' | sort | uniq -d | head -20
```

For each duplicate name: find ALL files containing it and confirm they are in different packages/modules (not just multiple files in the same package, which is normal).

```bash
grep -rn "def <name>\|function <name>\|class <name>" \
  --include="*.py" --include="*.ts" --include="*.go" 2>/dev/null \
  ! -path "*/node_modules/*" | head -10
```

## Step 3: Parallel implementations of the same concept

Look for evidence of the same concept implemented from scratch in multiple modules. Focus on the hotspot files:

```bash
# HTTP clients defined in multiple places
grep -rn "new.*HttpClient\|axios.create\|requests.Session\|http.NewClient" \
  --include="*.ts" --include="*.py" --include="*.go" 2>/dev/null \
  ! -path "*/node_modules/*" | head -10

# Config loaders in multiple places
grep -rn "loadConfig\|parseConfig\|readConfig\|getConfig" \
  --include="*.ts" --include="*.py" --include="*.go" 2>/dev/null \
  ! -path "*/node_modules/*" | head -10

# Error/exception classes defined multiple times
grep -rn "^class.*Error\|^class.*Exception\|^type.*Error " \
  --include="*.ts" --include="*.py" --include="*.go" 2>/dev/null \
  ! -path "*/node_modules/*" | head -15
```

## Severity guidance

| Finding | Severity | Confidence |
|---------|----------|-----------|
| Same function name exported from 4+ different modules | CRITICAL | HIGH |
| 3+ same-named utility files with overlapping symbols | HIGH | HIGH |
| Same function name in 2–3 different modules | HIGH | HIGH |
| Parallel HTTP/config/error implementation in 2 modules | MEDIUM | MEDIUM |
| Same-named file in 2 modules with non-overlapping symbols | LOW | HIGH |

## Finding format

```
#### [DUP] <title> — multiple locations | Confidence: HIGH
**Evidence:**
`find . -name "utils.*" | xargs wc -l` →
  `src/api/utils.ts (312 lines), src/worker/utils.ts (287 lines), src/scheduler/utils.ts (198 lines)`
`grep "^export function" src/api/utils.ts src/worker/utils.ts` →
  `src/api/utils.ts: export function retry`, `src/worker/utils.ts: export function retry`
**Impact:** Three teams are maintaining independent retry implementations — divergent behavior guaranteed.
**Recommendation:** Extract `src/shared/utils/retry.ts`; have all three import from there.
```
