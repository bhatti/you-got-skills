# Specialist: Knowledge Silos + Commit Quality

Review scope: single-author hotspot files (bus-factor 1), fix: commit ratio signals, large commits with poor atomicity, vague commit messages, AI-coauthored code concentration.

## Step 1: Knowledge silos in hotspot files

Use the pre-computed Knowledge Silos data. For each hotspot file with `top_author_pct > 0.75`:

```bash
# Verify the author concentration with actual git log
git log --pretty=format:"%an" -200 -- <file> 2>/dev/null \
  | sort | uniq -c | sort -rn | head -5

# Total commits analyzed
git log --oneline -200 -- <file> 2>/dev/null | wc -l
```

Calculate: `actual_pct = top_author_count / total_commits`. Only report if confirmed ≥ 75%.

Also check: is the top author still active in the last 90 days?

```bash
git log --pretty=format:"%an %ad" --date=short -200 -- <file> 2>/dev/null \
  | grep "^<top_author>" | tail -1
```

If the top author's last commit was > 90 days ago, upgrade severity (the silo is already orphaned).

## Step 2: Fix ratio and commit quality

```bash
# Compute fix ratio from last N commits
TOTAL=$(git log --oneline -1000 2>/dev/null | wc -l)
FIXES=$(git log --format="%s" -1000 2>/dev/null \
  | grep -icE "^fix[(:! ]|^bug[(:! ]|^hotfix[(:! ]|fixup\!|revert")
echo "Total: $TOTAL, Fix commits: $FIXES, Fix ratio: $(( FIXES * 100 / (TOTAL + 1) ))%"
```

```bash
# Large commits: touching >15 files
git log --oneline -500 2>/dev/null | while read hash msg; do
  count=$(git diff-tree --no-commit-id -r --name-only "$hash" 2>/dev/null | wc -l)
  if [ "$count" -gt 15 ]; then echo "$count files: $hash $msg"; fi
done | sort -rn | head -5
```

```bash
# Vague commit messages: < 15 characters in the subject line
git log --format="%H %s" -500 2>/dev/null \
  | awk 'length($0) - 41 < 15 {print}' | head -10
# (41 chars = hash + space)
```

```bash
# AI-coauthored commits
AI_COUNT=$(git log --format="%B" -1000 2>/dev/null \
  | grep -c "Co-Authored-By:.*Claude\|Co-Authored-By:.*Copilot\|Co-Authored-By:.*ChatGPT")
echo "AI-coauthored: $AI_COUNT / $TOTAL ($(( AI_COUNT * 100 / (TOTAL + 1) ))%)"
```

Show examples for each finding (commit hash + message), not just the count.

## Step 3: Commit atomicity check

```bash
# Find commits that touched >5 unrelated modules (cross-domain commits)
git log --oneline -200 2>/dev/null | while read hash msg; do
  files=$(git diff-tree --no-commit-id -r --name-only "$hash" 2>/dev/null)
  dirs=$(echo "$files" | sed 's|/[^/]*$||' | sort -u | wc -l)
  file_count=$(echo "$files" | wc -l)
  if [ "$dirs" -gt 5 ] && [ "$file_count" -gt 8 ]; then
    echo "$dirs dirs, $file_count files: $hash $msg"
  fi
done | head -5
```

## Severity guidance

| Finding | Severity | Confidence |
|---------|----------|-----------|
| Top-5 hotspot where single author wrote >80% AND last commit >90 days ago | CRITICAL | HIGH |
| Top-5 hotspot where single author wrote >80% (still active) | HIGH | HIGH |
| Fix: commit ratio > 40% | HIGH | HIGH |
| Top-10 hotspot where single author wrote >75% | HIGH | HIGH |
| Large commits (>15 files) > 10% of all commits | MEDIUM | HIGH |
| Fix: ratio 25–40% | MEDIUM | HIGH |
| Vague commits > 15% of all commits | MEDIUM | HIGH |
| AI-coauthored > 60% of commits | LOW | HIGH (informational) |

## Finding format

```
#### [SILO] <title> — <file> | Confidence: HIGH
**Evidence:**
`git log --pretty=format:"%an" -200 -- src/auth/handler.ts | sort | uniq -c | sort -rn` →
  `  182 alice@company.com`
  `   12 bob@company.com`
  `    6 carol@company.com`
Top author: 91% of 200 commits. Last commit: 2024-08-01 (41 days ago — still active).
**Impact:** Single point of failure for the auth handler — one team member leaving makes this module unmaintainable.
**Recommendation:** Schedule pair-programming or code walkthroughs. Add module-level README. Rotate ownership of next 3 features in this file.
```

```
#### [COMMIT] Fix ratio too high — <repo> | Confidence: HIGH
**Evidence:** `git log --format="%s" -1000 | grep -icE "^fix|^bug"` → `431 of 1000 commits (43%)`
Sample fix commits:
  - `a3f2c1b fix: null pointer in session handler` (2024-09-01)
  - `b9e4d22 fix: retry count off-by-one` (2024-08-28)
  - `c7f3a11 bug: wrong error code returned` (2024-08-25)
**Impact:** 43% reactive commits signals the team spends nearly half its time fixing regressions rather than building. Indicates missing test automation or architecture instability.
**Recommendation:** Add pre-merge test gate; track fix ratio as a team health metric.
```
