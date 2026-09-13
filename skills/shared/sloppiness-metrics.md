# Shared Reference: Sloppiness Metrics

Single source of truth for verbosity, mass, erosion, and cyclomatic complexity thresholds.
Referenced by: `ygs-codebase-audit/specialists/sloppiness.md`, `ygs-review-deep/specialists/maintainability.md`, `ygs-code-review/references/code-quality.md`, `ygs-learn/SKILL.md`.

---

## Metric Definitions and Benchmarks

| Metric | Formula | Established repos | AI-generated code |
|--------|---------|-------------------|-------------------|
| Verbosity | \|AST-Grep flagged lines ∪ clone lines\| / total LOC | 0.15 ± 0.06 | 0.33 ± 0.10 |
| Mass(f) | CC(f) × √SLOC(f) per function | — | — |
| Erosion | Σ(f: CC(f)>10) mass(f) / Σ(f) mass(f) | 0.31 ± 0.17 | 0.68 ± 0.20 |

Source: "Measuring Code Sloppiness" (earendil.com). AI-generated code is on average **2× more verbose** and has **2× the erosion** of established repos.

---

## Cyclomatic Complexity (CC) Thresholds

CC = number of linearly independent paths through a function. Count: `if / else if / for / while / case / catch / && / ||` — each adds 1, start at 1.

| CC Range | Risk | Action |
|----------|------|--------|
| 1–5 | Low — simple, easy to test | No action |
| 6–10 | Medium — each branch needs a test | Monitor |
| > 10 | High — hard to test, high defect probability | Flag **SHOULD** split |
| > 15 | Very high — virtually untestable | Flag **MUST** split |

---

## Metric Thresholds (for reporting)

| Metric | Healthy | Warning | High |
|--------|---------|---------|------|
| Verbosity | < 0.20 🟢 | 0.20–0.30 🟡 | > 0.30 🔴 |
| Erosion | < 0.40 🟢 | 0.40–0.55 🟡 | > 0.55 🔴 |
| High-mass functions (CC > 10) | 0 🟢 | 1–3 🟡 | 4+ 🔴 |
| Churn × CC hotspots | 0 🟢 | 1–2 🟡 | 3+ 🔴 |

**False positive guard**: Only flag Verbosity or Erosion when **tool output confirms the value**. Do not estimate or interpolate.

---

## Verbosity Anti-Pattern Catalogue

These patterns are the primary contributors to high verbosity ratios. Flag only when clearly present in code — never infer.

### 1. Trivial Delegators
A method whose entire body is a single forwarding call with no transformation, validation, or side effect.
```python
# BAD — trivial delegator
def get_user_name(self):
    return self.user_service.get_user_name()

# OK — adds validation or transforms
def get_user_name(self):
    name = self.user_service.get_user_name()
    return name.strip() if name else "unknown"
```

### 2. Wrapper-of-Wrapper
A class that wraps a class that wraps a class, adding no logic at any layer.
```python
# BAD — no value added at MyApiClient layer
class MyApiClient:
    def __init__(self): self.inner = ApiClientWrapper()
    def fetch(self, url): return self.inner.fetch(url)
```

### 3. Boilerplate Getters/Setters
Auto-generated or manually written getter/setter for every field, including fields that are never accessed externally.

### 4. Duplicated Guard Blocks
The same `if x is None: raise ValueError(...)` or `if not authorized: return 403` pattern repeated at multiple call sites instead of centralizing in the callee.

### 5. Shallow Pass-Through Functions
A function that only renames or reorders arguments to call one other function — adds indirection without depth.

### 6. Redundant Type Casts / Assertions
The same `str(x)`, `int(x)`, or `assert isinstance(x, T)` repeated across multiple callers when the callee could enforce the type itself.

---

## Tooling

### lizard — CC + SLOC per function (preferred)
```bash
# Install
pip install lizard

# Run on repo — CSV output for scripting
lizard . --csv 2>/dev/null | head -5
# Output columns: NLOC, CC, token_count, PARAM, length, location, filepath, function_name, long_name, start, end

# Top high-CC functions
lizard . --csv 2>/dev/null \
  | awk -F',' 'NR>1 && $2>10 {printf "CC=%s SLOC=%s mass=%.1f %s::%s\n", $2, $1, $2*sqrt($1), $7, $8}' \
  | sort -t= -k2 -rn | head -20

# Compute Erosion
lizard . --csv 2>/dev/null \
  | awk -F',' 'NR>1 && $1>0 && $2>0 {
      mass=$2*sqrt($1); total+=mass;
      if($2>10) high+=mass
    } END {
      if(total>0) printf "Erosion=%.2f (high_mass=%.1f / total_mass=%.1f)\n", high/total, high, total
    }'
```

Supported languages: Python, Go, Rust, JavaScript, TypeScript, C, C++, Java, Swift, and more.

### jscpd — Clone/duplicate block detection (for Verbosity numerator)
```bash
# Install
npm install -g jscpd

# Run — JSON report
jscpd . --min-lines 6 --min-tokens 50 --reporters json \
  --output /tmp/jscpd-out --ignore "**/.git/**,**/vendor/**,**/node_modules/**" 2>/dev/null

# Extract duplicate line count from report
python3 -c "
import json, sys
try:
    r=json.load(open('/tmp/jscpd-out/jscpd-report.json'))
    print('Clone lines:', r.get('statistics',{}).get('total',{}).get('duplicatedLines',0))
    print('Clone pct:',   r.get('statistics',{}).get('total',{}).get('percentage',0),'%')
except: print('jscpd report not found')
"
```

### Fallback (no lizard/jscpd available)
```bash
# Large function heuristic (proxy for CC > 10: function body > 40 lines)
grep -rn "^def \|^func \|^function \|^pub fn \|^fn " \
  --include="*.py" --include="*.go" --include="*.js" --include="*.ts" --include="*.rs" \
  . 2>/dev/null | wc -l   # total functions

# Count functions > 40 lines (rough CC > 10 proxy)
awk '/^def |^func |^function |^pub fn |^fn /{if(body>40)print FILENAME":"start" body="body" lines"; start=NR; body=0} {body++}' \
  $(find . -name "*.py" -o -name "*.go" 2>/dev/null | head -50) 2>/dev/null | head -20
```

---

## Churn × Complexity Hotspot (Risky Quadrant)

Files that are both **frequently changed** (top hotspots by commit count) AND **high complexity** (contain CC>10 functions) are the highest-defect-risk files. Changes to these files are most likely to introduce bugs.

```bash
# Step 1: get top hotspot files from pre-computed data or:
git log --format='' --name-only -500 2>/dev/null | sort | uniq -c | sort -rn | head -20 | awk '{print $2}' > /tmp/hotspots.txt

# Step 2: find which hotspot files have high-CC functions
lizard . --csv 2>/dev/null \
  | awk -F',' 'NR>1 && $2>10 {print $7}' | sort -u > /tmp/highcc_files.txt

# Step 3: intersection = risky quadrant
comm -12 <(sort /tmp/hotspots.txt) <(sort /tmp/highcc_files.txt)
```
