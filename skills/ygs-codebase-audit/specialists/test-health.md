# Specialist: Test Health

Review scope: production files with no test coverage, brittle tests that change more than production code, disabled/skipped tests accumulating technical debt.

## Step 1: Verify untested production files

Use the pre-computed `untested_files` list from the Repository Analysis Data. For each file listed, **actively search for its test** before flagging — the pre-computed data may have false positives.

```bash
# For each candidate untested production file:
PROD_FILE="<file_path>"
BASENAME=$(basename "$PROD_FILE" | sed 's/\..*//')

# Search for test counterpart (all common naming conventions)
find . \( \
  -name "test_${BASENAME}.*" -o \
  -name "${BASENAME}_test.*" -o \
  -name "${BASENAME}.test.*" -o \
  -name "${BASENAME}.spec.*" -o \
  -name "${BASENAME}Test.*" -o \
  -name "${BASENAME}Spec.*" \
\) ! -path "*/.git/*" ! -path "*/node_modules/*" 2>/dev/null

# Also search in common test directories
find . -path "*/test*/${BASENAME}*" -o -path "*/spec*/${BASENAME}*" \
  ! -path "*/.git/*" ! -path "*/node_modules/*" 2>/dev/null | head -5
```

Only flag as "untested" if BOTH searches return empty. If a test file exists anywhere in the project for this module, do NOT flag it.

Also check the file's commit history — if it was added in the last 3 commits, it may be too new to have tests yet (flag as MEDIUM not HIGH):

```bash
git log --oneline -3 -- <prod_file> 2>/dev/null
```

## Step 2: Brittle tests

From the pre-computed `brittle_test_files` list, verify the churn asymmetry:

```bash
# Get exact commit count for the test file
git log --oneline -- <test_file> 2>/dev/null | wc -l

# Get exact commit count for inferred production counterpart
PROD=$(echo "<test_file>" | sed 's|test_||; s|_test\.||; s|\.test\.|\.|; s|\.spec\.|\.|; s|Test\b||')
git log --oneline -- "$PROD" 2>/dev/null | wc -l

# Show last 5 test-only change commits (changed test but not prod)
git log --oneline -50 -- <test_file> 2>/dev/null | while read hash msg; do
  if git diff-tree --no-commit-id -r --name-only "$hash" 2>/dev/null | grep -q "$PROD"; then
    echo "BOTH: $hash $msg"
  else
    echo "TEST-ONLY: $hash $msg"
  fi
done | grep "TEST-ONLY" | head -5
```

Only flag if: (a) test churn ≥ 2× prod churn, AND (b) you found ≥ 3 test-only change commits.

## Step 3: Skip/disabled test markers

```bash
# Count disabled tests across the codebase
grep -r \
  "skip\|@pytest.mark.skip\|@pytest.mark.xfail\|it\.skip\|describe\.skip\
\|xit\b\|xdescribe\|@Ignore\|t\.Skip\b\|TODO.*test\|FIXME.*test" \
  --include="*test*" --include="*spec*" \
  ! -path "*/.git/*" ! -path "*/node_modules/*" ! -path "*/vendor/*" \
  -l 2>/dev/null | head -20

# Get the count
SKIP_COUNT=$(grep -rc \
  "skip\|@pytest.mark.skip\|it\.skip\|xit\b\|@Ignore\|t\.Skip\b" \
  --include="*test*" --include="*spec*" \
  ! -path "*/.git/*" ! -path "*/node_modules/*" 2>/dev/null \
  | awk -F: '{sum += $2} END {print sum}')
echo "Total skip markers: $SKIP_COUNT"

# Show the worst offenders (files with most skips)
grep -rc "skip\|@pytest.mark.skip\|it\.skip\|xit\b\|@Ignore" \
  --include="*test*" --include="*spec*" \
  ! -path "*/.git/*" ! -path "*/node_modules/*" 2>/dev/null \
  | grep -v ":0$" | sort -t: -k2 -rn | head -5
```

## Step 4: Coverage gaps in hotspot files

For the top-10 hotspot files by churn, cross-reference against the untested list:

```bash
# For each top hotspot, check if it has a test AND if it recently changed without test co-changes
git log --oneline -20 -- <hotspot_file> 2>/dev/null | while read hash msg; do
  changed=$(git diff-tree --no-commit-id -r --name-only "$hash" 2>/dev/null)
  has_test=$(echo "$changed" | grep -ciE "test|spec")
  if [ "$has_test" -eq 0 ]; then echo "no-test: $hash $msg"; fi
done | wc -l
```

A hotspot file changed frequently with NO test co-changes is a HIGH finding even if a test file exists — it means the tests aren't being updated alongside the code.

## Severity guidance

| Finding | Severity | Confidence |
|---------|----------|-----------|
| Top-5 hotspot file with NO test file found anywhere | CRITICAL | HIGH |
| Hotspot in top-10 changed 15+ times with zero test co-changes | HIGH | HIGH |
| Test file churn ≥ 3× production counterpart (≥5 test-only commits) | HIGH | HIGH |
| >20 skip markers across codebase | HIGH | HIGH |
| Hotspot changed 8-14 times with < 20% test co-change rate | MEDIUM | HIGH |
| Test churn 2–3× production (3-5 test-only commits) | MEDIUM | MEDIUM |
| 10-20 skip markers | MEDIUM | HIGH |

## Finding format

```
#### [TEST] <title> — <file> | Confidence: HIGH
**Evidence:**
`git log --oneline -- src/auth/handler.ts | wc -l` → `34`
`find . -name "*handler*test*" -o -name "*test*handler*" | grep -v node_modules` → (empty)
`git log --oneline -20 -- src/auth/handler.ts | grep -c "no-test"` → `18 of 20 commits had no test co-change`
**Impact:** Core auth handler changed 34 times with no test coverage — regressions invisible.
**Recommendation:** Add unit tests for the top-5 exported functions; add CI gate requiring test co-changes for this file.
```
