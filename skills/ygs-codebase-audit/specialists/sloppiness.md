# Specialist: Sloppiness — Verbosity, Mass, and Erosion

Review scope: structural verbosity and cyclomatic complexity accumulated across commits — patterns that grow silently with AI-generated or rushed code. Produces three benchmark-calibrated metrics: Verbosity ratio, Erosion score, and a Churn × Complexity risky-quadrant list.

**Reference:** Read `~/.claude/skills/you-got-skills/skills/shared/sloppiness-metrics.md` first — it defines all formulas, thresholds, verbosity anti-pattern catalogue, and tooling invocations. Do not duplicate that content here; apply it.

---

## ⚠️ No false positives

Every metric value must come from actual tool output. If lizard or jscpd are unavailable, use the fallback grep heuristics defined in `shared/sloppiness-metrics.md` and label findings `Confidence: MEDIUM`. Do not estimate or interpolate metric values.

---

## Step 1: LOC Baseline

```bash
# Preferred: tokei gives SLOC (strips comments/blanks)
tokei . --output json 2>/dev/null | python3 -c "
import json,sys
d=json.load(sys.stdin)
total=sum(v.get('code',0) for k,v in d.items() if k!='Total')
print('Total SLOC:', total)
" 2>/dev/null || echo "tokei not available"

# Fallback: wc -l on source files
find . \( -name "*.py" -o -name "*.go" -o -name "*.rs" -o -name "*.ts" -o -name "*.js" \) \
  ! -path "*/.git/*" ! -path "*/vendor/*" ! -path "*/node_modules/*" ! -path "*/dist/*" \
  2>/dev/null | xargs wc -l 2>/dev/null | tail -1
```

Record `TOTAL_SLOC` for use in the Verbosity formula.

---

## Step 2: Mass and Erosion (lizard)

```bash
# Check if lizard is available
lizard --version 2>/dev/null || echo "lizard not installed — pip install lizard"

# Top 20 highest-mass functions (CC × √SLOC)
lizard . --csv 2>/dev/null \
  | awk -F',' 'NR>1 && $1>0 && $2>0 {
      mass=$2*sqrt($1);
      printf "mass=%.1f CC=%s SLOC=%s %s::%s\n", mass, $2, $1, $7, $8
    }' \
  | sort -t= -k2 -rn | head -20

# Erosion score
lizard . --csv 2>/dev/null \
  | awk -F',' 'NR>1 && $1>0 && $2>0 {
      mass=$2*sqrt($1); total+=mass;
      if($2>10) high+=mass
    } END {
      if(total>0) printf "Erosion=%.2f (high_mass=%.1f / total_mass=%.1f)\n", high/total, high, total
      else print "No functions found"
    }'

# Count of functions with CC > 10 and CC > 15
lizard . --csv 2>/dev/null \
  | awk -F',' 'NR>1 {
      if($2>15) c15++
      else if($2>10) c10++
    } END {printf "CC>10: %d  CC>15: %d\n", c10+c15, c15}'
```

Record: `EROSION_SCORE`, `HIGH_MASS_FN_COUNT` (CC > 10), `CRITICAL_MASS_FN_COUNT` (CC > 15).

---

## Step 3: Clone Detection (jscpd — Verbosity numerator)

```bash
# Check availability
jscpd --version 2>/dev/null || echo "jscpd not installed — npm install -g jscpd"

jscpd . --min-lines 6 --min-tokens 50 --reporters json \
  --output /tmp/jscpd-out \
  --ignore "**/.git/**,**/vendor/**,**/node_modules/**,**/dist/**,**/build/**" \
  2>/dev/null

python3 -c "
import json
try:
    r=json.load(open('/tmp/jscpd-out/jscpd-report.json'))
    s=r.get('statistics',{}).get('total',{})
    print('Clone lines:', s.get('duplicatedLines',0))
    print('Clone pct:',  round(s.get('percentage',0),1), '%')
    # list top 3 clone instances
    dupes=r.get('duplicates',[])[:3]
    for d in dupes:
        print(' ', d.get('firstFile',{}).get('name','?'), 'x', d.get('secondFile',{}).get('name','?'))
except Exception as e:
    print('jscpd report unavailable:', e)
" 2>/dev/null
```

Record `CLONE_LINES`.

---

## Step 4: Verbosity Pattern Grep (lightweight supplement)

Run these grep patterns to count verbosity-indicator lines. Each pattern targets one anti-pattern from `shared/sloppiness-metrics.md`.

```bash
# Trivial delegators (Python: method body is a single return self.x.y() call)
grep -rn "^\s*def \w\+(" --include="*.py" -l 2>/dev/null \
  | xargs grep -l "^\s*return self\.\w\+\.\w\+(" 2>/dev/null | wc -l
echo "Python files with trivial-delegator pattern: above"

# Go: trivial delegator (method body = single return call)
grep -rn -A2 "^func (" --include="*.go" 2>/dev/null \
  | grep -c "^\s*return [a-z]\+\.[A-Z][a-zA-Z]*(" 2>/dev/null || true
echo "Go trivial delegator lines: above"

# Boilerplate getter/setter pairs (Python @property + .setter for same name)
grep -rn "@property" --include="*.py" 2>/dev/null | wc -l
echo "Python @property decorators (check if all are necessary): above"

# Duplicated guard: same nil/None check at 3+ call sites
grep -rn "if.*== nil\|if.*is None\|if.*!= nil\|if.*is not None" \
  --include="*.go" --include="*.py" --include="*.ts" 2>/dev/null \
  | awk -F':' '{print $2}' | sort | uniq -c | sort -rn | awk '$1>=3' | head -10
echo "Repeated guard patterns (3+ occurrences): above"
```

These counts contribute to pattern detection but are **not included in the Verbosity ratio formula** (which requires AST-Grep or clone tool output). Label grep-based observations as `Confidence: MEDIUM`.

---

## Step 5: Verbosity Ratio

Compute after Steps 1–3 are complete:

```
VERBOSITY = (CLONE_LINES + verbosity_grep_flagged_lines) / TOTAL_SLOC
```

Where:
- `CLONE_LINES` = from jscpd output (Step 3)
- `verbosity_grep_flagged_lines` = conservative estimate from Step 4 grep counts (each trivial-delegator method ≈ 3 lines; use 0 if uncertain)
- `TOTAL_SLOC` = from Step 1

If jscpd is unavailable, compute using clone pct × TOTAL_SLOC as approximation; label `Confidence: MEDIUM`.

---

## Step 6: Churn × Complexity Risky Quadrant

Cross-reference the pre-computed `## Hotspot Analysis` data (Phase 1d) with high-CC files from Step 2.

```bash
# Extract hotspot file paths from pre-computed data (adjust grep for actual format)
# Assumes hotspot lines look like: "  42  src/api/handler.go"
grep -E "^\s+[0-9]+" <<'HOTSPOT_DATA'
[PASTE hotspot lines from pre-computed data here — or run:]
HOTSPOT_DATA

# Get high-CC files from Step 2 output
lizard . --csv 2>/dev/null \
  | awk -F',' 'NR>1 && $2>10 {print $7}' | sort -u > /tmp/highcc_files.txt

# Get top-20 hotspot files
git log --format='' --name-only -500 2>/dev/null \
  | grep -E "\.(py|go|rs|ts|js)$" | sort | uniq -c | sort -rn \
  | head -20 | awk '{print $2}' > /tmp/hotspot_files.txt

# Risky quadrant: intersection
RISKY=$(comm -12 <(sort /tmp/hotspot_files.txt 2>/dev/null) \
              <(sort /tmp/highcc_files.txt 2>/dev/null) 2>/dev/null)
echo "Risky quadrant files (high churn + high CC):"
echo "${RISKY:-none}"
```

---

## Severity Guidance

| Finding | Severity | Confidence |
|---------|----------|-----------|
| Erosion > 0.55 (tool-computed) | HIGH | HIGH |
| Erosion 0.40–0.55 (tool-computed) | MEDIUM | HIGH |
| Verbosity > 0.30 (tool-computed) | HIGH | HIGH |
| Verbosity 0.20–0.30 (tool-computed) | MEDIUM | HIGH |
| 3+ risky-quadrant files (high churn + CC > 10) | HIGH | HIGH |
| CC > 15 functions exist (tool-computed) | HIGH | HIGH |
| CC 10–15 functions exist, count > 3 | MEDIUM | HIGH |
| Repeated guard patterns (3+ sites, grep-confirmed) | LOW | MEDIUM |
| High @property / getter-setter count (grep-confirmed) | LOW | MEDIUM |

---

## Finding Format

```
#### [SLOP] High Erosion Score — codebase-wide | Confidence: HIGH
**Evidence:**
`lizard . --csv | awk '...'` →
  `Erosion=0.71 (high_mass=1842.3 / total_mass=2596.1)`
  `Top mass: CC=18 SLOC=95 src/api/processor.go::processRequest`
**Benchmark:** Established repos: 0.31 ± 0.17 · AI-generated: 0.68 ± 0.20 · This repo: 0.71 🔴
**Impact:** 71% of codebase complexity is concentrated in functions with CC > 10, making them
  hard to test and high-probability defect sources. These functions are also the ones changing most often.
**Recommendation:** Start with the top-3 highest-mass functions above. Each CC > 10 function
  should be split at logical seams until CC ≤ 6. Run lizard after each split to verify progress.
```

```
#### [SLOP] Verbosity Ratio Above Threshold | Confidence: HIGH
**Evidence:**
`jscpd ...` → `Clone lines: 847 (8.4% of SLOC)`
`TOTAL_SLOC: 10100`
`Verbosity = (847 + 0) / 10100 = 0.084`  ← (or 0.28 if high)
**Benchmark:** Established repos: 0.15 ± 0.06 · AI-generated: 0.33 ± 0.10 · This repo: X 🟢/🟡/🔴
**Impact:** [describe impact if above threshold]
**Recommendation:** [if above threshold: identify top clone clusters from jscpd report and extract to shared module]
```
