---
name: ygs-changelog
description: Generate a structured changelog from git history and completed task files. Use for release notes, sprint summaries, or CHANGELOG.md updates.
argument-hint: "[--since <tag-or-date>] [--format release|sprint|markdown]"
---

# Changelog

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — you own the changelog quality.

Generate a human-readable changelog from git history and task completion records.

## Step 1: Determine scope

If `--since` is provided, use that as the lower bound.
Otherwise, detect the last release tag:

```bash
git describe --tags --abbrev=0 2>/dev/null || echo "(no prior tag — using all commits)"
```

## Step 2: Gather signals

**From git:**
```bash
git log $(git describe --tags --abbrev=0 2>/dev/null)..HEAD --oneline --no-merges 2>/dev/null
```

**From completed tasks:**
```bash
ls -t tasks/done/*.md 2>/dev/null | head -50
```
Read each done task for: title, type (feature/fix/chore), and one-line summary.

**From PRs (if GitHub):**
```bash
gh pr list --state merged --json number,title,labels,mergedAt --limit 50 2>/dev/null \
  | jq 'sort_by(.mergedAt) | reverse'
```

## Step 3: Classify entries

Group into conventional changelog categories:

| Category | Signals |
|----------|---------|
| **Added** | New features, new endpoints, new commands |
| **Changed** | Modified behavior, updated defaults, renamed things |
| **Fixed** | Bug fixes, error handling improvements |
| **Removed** | Deleted features, deprecated and removed APIs |
| **Security** | Dependency updates fixing CVEs, auth changes |
| **Performance** | Measurable speed or resource improvements |
| **Chore** | Dependency bumps, CI changes, tooling (omit from user-facing changelogs) |

Deduplicate: commits and task records often describe the same change. Merge them into one entry.

## Step 4: Write the changelog

**Sprint summary format (`--format sprint`):**
```
Sprint N — <start> to <end>
Added: X features
Fixed: Y bugs
<bullet per significant item>
```

**Release format (`--format release` or default):**
```markdown
## [<version>] — <date>

### Added
- Description of new capability (#PR or task ref)

### Fixed
- Description of fix (#PR or task ref)

### Security
- Dependency: <package> bumped from X to Y (CVE-YYYY-NNNNN)
```

Output to stdout by default. If `CHANGELOG.md` exists at repo root, append the new section at the top (below the title, above the previous release).

## Step 5: Verify

Check that the generated changelog:
- Covers the full scope (no significant commit without an entry)
- Has no entries that duplicate each other
- Has working PR/task references

## Completion

Report **DONE** with a summary of entries generated.

Suggest: `/ygs-ship` to tag the release after changelog is committed.
