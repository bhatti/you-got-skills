---
name: ygs-learn
description: Capture and surface operational learnings across sessions — failures, surprises, patterns that should inform future work. Use after incidents, surprising bugs, or when a pattern keeps recurring.
---

# Learn

Persistent knowledge capture that feeds back into future sessions. Unlike `/ygs-retro` (which reviews a period of work), `/ygs-learn` captures a single atomic learning as it happens.

When invoked with a combined post-merge prompt (Phase 0 + Phase 1), first run the PR health check (Phase 0), then extract learnings (Phase 1).

## Phase 0: PR Health Check (single PR)

When the prompt includes a "PR Data" section with pre-computed fields, run a quick health check **before** extracting learnings. This is a single-PR check — do NOT do cross-PR pattern analysis or frequency-based findings. Omit any dimension where there is nothing to flag.

For each dimension, write 1–3 bullet points under `## PR Health Analysis`:

**Spec Coverage**
- Check `has_acceptance_criteria` in the PR data. If `true`, note the AC briefly.
- If `false`, read the issue excerpt — does it describe testable behavior? If yes, note "semantic AC present".
- Flag "AC missing" only if the issue description has NO testable outcome at all.

**Design Decisions**
- Did this PR make a significant architectural choice? (new pattern, new dependency, new abstraction)
- Check: `ls docs/adr/ 2>/dev/null | head -5`
- If a decision was made and no ADR exists for it, recommend one with a 2-line summary of what to capture.

**Security & SRE**
- Scan `file_paths` for: `auth`, `rbac`, `iam`, `credential`, `secret`, `permission`, `token`, `oauth`
- If present: was security reviewed? (check human comments for security keywords)
- Was a rollback plan or feature flag mentioned? Were metrics/logging/alerting added for new code paths?

**Review Quality** — only flag HIGH blast-radius issues (same taxonomy as `ygs-pr-audit`)
- Check `rubber_stamp_approvers`. If non-empty AND files touch auth/billing/infra → flag.
- Check `substantive_human_comment_count`. If 0 on a bot-authored PR → flag.
- Low-risk silent approval is acceptable — do not flag.

**CI Health**
- Count "Build #" in CI bot comments. Flag only if ≥5 iterations (indicates missing pre-push checks).

## Step 1: Capture the learning

Ask (or infer from context):
- **What happened?** — The specific event, bug, or surprise
- **Why does it matter?** — What would go wrong next time without this knowledge
- **Category:** Edge Case | Integration Gotcha | Performance Cliff | Security Trap | Process Friction | Domain Rule | Tooling Quirk

## Step 2: Check for duplicates

```bash
find docs/learnings/ -name "*.md" 2>/dev/null | xargs grep -l "<relevant keyword>" 2>/dev/null
```

If a similar learning exists: reinforce it (add date + additional evidence) rather than duplicating.

## Step 3: Write the learning

```bash
mkdir -p docs/learnings
```

Write to `docs/learnings/YYYY-MM-DD-<slug>.md`:

```markdown
# <Title>

**Category:** <category>
**Date:** YYYY-MM-DD
**Source:** <what triggered this — incident, review finding, spike result, etc.>

## Learning

<1-3 sentences: the rule or pattern>

## Evidence

<What happened that taught us this>

## Application

<When to apply this in future work — specific trigger conditions>
```

## Step 4: Surface relevant learnings (when invoked without a new learning)

If the user asks "what do we know about X?" or invokes without new context:

```bash
grep -rl "<keyword>" docs/learnings/ 2>/dev/null
```

Present matching learnings sorted by relevance. Note any that may be stale (older than 6 months — verify still applies).

## Step 4a: Extract mode (after PR merge — invoked with `--extract <pr-url>`)

When a PR has been merged and review feedback is available, extract learnings from that feedback automatically:

1. Fetch PR comments via `gh pr view <pr-url> --comments` (or Bitbucket REST equivalent)
2. For each comment thread: assess whether it reveals a recurring pattern, a gotcha, or a surprise that would apply to future work
3. Skip comments that are purely stylistic, already captured, or specific to the PR with no generalizability
4. For each actionable learning: run Steps 2-3 above (deduplicate, then write)
5. Report: N learnings extracted, M skipped (already existed or not generalizable)

## Step 4b: Audit mode (invoked with `--audit`)

Scan learnings for staleness and quality:

```bash
find docs/learnings/ -name "*.md" -mtime +90 2>/dev/null
```

For each learning older than 90 days:
- Check whether the evidence still applies (the code/pattern it references may have changed)
- If stale: update with new evidence, or mark as superseded and delete
- Report: N reviewed, M updated, K deleted

## Step 5: Completion

Report **DONE** with:
- Learning captured (or surfaced)
- File path
- Related learnings (if any exist in the same category)

Suggest: `/ygs-retro` for broader reflection, or `/ygs-investigate` if the learning came from a bug.
