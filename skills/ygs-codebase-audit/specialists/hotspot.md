# Specialist: Hotspot + Temporal Coupling

Review scope: files that change too often, change together across module boundaries, or represent concentrated blast radius.

## Step 1: Build ranked hotspot list

Use the pre-computed Hotspot Analysis from the Repository Analysis Data. If it is missing or sparse, run:

```bash
git log --name-only --pretty=format: -1000 2>/dev/null \
  | grep -v "^$" | sort | uniq -c | sort -rn | head -30
```

Compute risk score for each file: `risk = change_count × (2 if in temporal coupling else 1)`

Rank top-20 by risk score. Record each file's:
- change_count and what percentage of total commits it represents
- Whether it appears in any temporal coupling pair

## Step 2: Verify each candidate before flagging

For each hotspot candidate above the severity threshold:

```bash
# Confirm file still exists and is a production code file (not generated/lock/changelog)
ls -la <file> 2>/dev/null && head -3 <file> 2>/dev/null

# Get exact commit count with evidence
git log --oneline -- <file> 2>/dev/null | wc -l

# Sample recent commit messages to understand why it changes so often
git log --oneline -10 -- <file> 2>/dev/null
```

Skip if: file does not exist, is a lockfile (`package-lock.json`, `go.sum`, `Cargo.lock`), is a generated file (`*.pb.go`, `*_generated.*`), or is a changelog/migration file.

## Step 3: Temporal coupling deep-dive

For each coupling pair in the pre-computed data with confidence ≥ 0.6:

```bash
# Verify the two files are in DIFFERENT top-level directories (intra-package coupling is normal)
echo "File A dir: $(dirname <file_a>)"
echo "File B dir: $(dirname <file_b>)"

# Show 3 example commits where both changed together
git log --oneline -200 2>/dev/null | while read hash msg; do
  files=$(git diff-tree --no-commit-id -r --name-only "$hash" 2>/dev/null)
  if echo "$files" | grep -q "<file_a>" && echo "$files" | grep -q "<file_b>"; then
    echo "$hash $msg"
  fi
done | head -3
```

Only report coupling where (1) files are in different top-level directories, (2) you found ≥ 3 actual co-change examples in git log, (3) both files currently exist.

## Severity thresholds

| Condition | Severity | Confidence |
|-----------|----------|-----------|
| Changed in >15% of commits AND in temporal coupling with conf>0.6 | CRITICAL | HIGH |
| Changed in >10% of commits OR temporal coupling conf>0.7 (cross-module) | HIGH | HIGH |
| Changed in >5% of commits | MEDIUM | HIGH |
| Temporal coupling conf 0.5–0.7 cross-module | MEDIUM | MEDIUM |

## Finding format

```
#### [HOTSPOT] <short title> — <file:line_range_if_relevant> | Confidence: HIGH
**Evidence:** `git log --oneline -- <file> | wc -l` → `47 commits (4.7% of 1000)`
**Coupling:** Co-changes with `<other_file>` in 12/47 commits (conf=0.85)
**Sample commits:**
  - `a3f2c1b fix: retry logic in auth handler` (2024-01-15)
  - `b9e4d22 refactor: move session state` (2024-01-08)
**Impact:** Concentrated churn signals hidden complexity or missing abstraction boundary.
**Recommendation:** [specific refactor suggestion]
```
