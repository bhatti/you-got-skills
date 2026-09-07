---
name: ygs-retro
description: Retrospective on recent work — what went well, what didn't, what to change. Use after completing a feature or sprint.
---

# Retrospective

## Step 1: Gather context

Review recent work:

```bash
git log --oneline --since="1 week ago" 2>/dev/null || git log --oneline -20
```

Check completed tasks:

```bash
ls tasks/done/*.md 2>/dev/null
```

Read recent PRDs/TRDs for what was planned vs delivered.

## Step 2b: Git Analytics (quick quantitative signal)

Before asking the human, collect git signal to anchor the retrospective in data:

```bash
SINCE="${RETRO_SINCE:-1 week ago}"
# Commit type ratios
TOTAL=$(git log --format="%s" --since="$SINCE" 2>/dev/null | wc -l | tr -d ' ')
FIXES=$(git log --format="%s" --since="$SINCE" 2>/dev/null | grep -cE "^fix[(:!]|^bug[(:!]|^hotfix" || echo 0)
FEATS=$(git log --format="%s" --since="$SINCE" 2>/dev/null | grep -cE "^feat[(:!]" || echo 0)
if [ "$TOTAL" -gt 0 ]; then
    echo "Total commits: $TOTAL | feat ratio: $((FEATS * 100 / TOTAL))% | fix ratio: $((FIXES * 100 / TOTAL))%"
else
    echo "No commits in the period"
fi

# Top-5 hotspot files (changed most often this period)
echo "--- Hotspot files ---"
git log --name-only --pretty=format: --since="$SINCE" 2>/dev/null \
  | grep -v "^$" | sort | uniq -c | sort -rn | head -5

# Large commits (many files — risky for regressions)
echo "--- Large commits (>10 files) ---"
git log --format="%h %s" --since="$SINCE" 2>/dev/null | while read hash msg; do
    count=$(git show --name-only "$hash" 2>/dev/null | tail -n +6 | grep -v "^$" | wc -l | tr -d ' ')
    [ "$count" -gt 10 ] && echo "  $hash ($count files): $msg"
done | head -5
```

Flag in the retrospective narrative:
- fix: ratio >40% → "Team is reactive — shipping debt faster than features"
- Same file changed >3× in one week → "Potential hotspot: consider refactor"
- Any commit touching >15 files → "Large commit — review risk and test coverage"

For a deeper analysis of the last 1000 commits across all teams, use `ygs-codebase-audit` instead.

## Step 2: Ask the human

The artifacts show what happened; only the human observed friction, judgment calls, and frustration. After reviewing artifacts, ask 3-4 targeted questions — tailor them to what you found, not generic prompts.

**Always ask:**
- "What was the most frustrating part of this work?"

**Pick 2-3 based on evidence:**
- If many deviations: "The design had N deviations — did the design miss something fundamental, or were these natural adjustments?"
- If scope guardrails fired: "Tasks exceeded scope — was the breakdown too optimistic, or did requirements shift?"
- If reviews were slow or contentious: "Were code review findings useful, or did they feel like busywork?"
- If things went smoothly: "What made this go smoothly — what should we repeat?"

The human's answers are the highest-signal input. Weight them accordingly in Step 3.

## Step 3: Analyze patterns

Assess across these dimensions, informed by both artifacts and human answers:

### Estimation accuracy
- Estimated vs actual file/line counts — which tasks were under/oversized and why?
- Were entire categories of work (migrations, test infra, integration glue) missed?

To calibrate estimation accuracy with data, pull velocity from tracker:

Follow `~/.claude/skills/you-got-skills/skills/shared/tracker.md` — query "Recent velocity" to get story points completed in the last 1-2 sprints. Compare against estimates made at sprint start.

### Design quality
- Which decisions held up? Which required deviations? What does the deviation pattern suggest?

### Task breakdown quality
- Were tasks right-sized in practice? Any that should have been split or merged?
- Was dependency ordering correct?

### Process friction
- Where did work stall? Where was intervention most needed?
- Which process steps caught real problems vs added overhead?

### Recurring patterns
- Same issue across multiple tasks = systemic problem, not bad luck
- Areas of the codebase that consistently cause problems?

## Step 4: Recommendations

Produce 2-3 specific, actionable changes:
- **Keep doing:** Practices that worked well
- **Start doing:** New practices to try
- **Stop doing:** Practices causing friction

Be specific: not "communicate better" — instead "add acceptance criteria before implementation starts". One task needing extra effort is noise; the same pattern across three tasks is signal.

## Step 5: Incidents

If any incidents occurred during the period, use `references/postmortem-template.md` for structured post-mortem analysis with Five Whys and action items.

## Step 6: Extract learnings from code reviews

For baseline test health, follow `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.

If PR reviews or code review feedback exists for the period:

```bash
find tasks/done/ -name "*.md" -exec grep -l "review\|feedback\|deviation" {} \; 2>/dev/null
```

For each piece of review feedback that was addressed:
1. Categorize: Edge Case | Bug in Fix | Code Pattern | Scope | Security | Performance | Test Quality | Style
2. Assess if it's generalizable (would apply to future work, not just this instance)
3. If generalizable: extract as a learning with the pattern and why it matters

Present learnings one at a time. For each, ask: save to project memory? skip? rephrase?

Learnings that appear across multiple reviews get higher signal — note the count. If a learning already exists in memory, reinforce it (note additional evidence) rather than duplicating.

## Step 7: Completion

Report **DONE** with the retrospective summary.
Suggest saving as `docs/retro/YYYY-MM-DD.md` if the team wants to track retros.
