# Team Intelligence Init

Common Step 1 for ygs-standup, ygs-risk-scan, and ygs-sprint-plan.
Reference this instead of repeating the block.

```bash
cat .ygs/tracker.yml 2>/dev/null || echo "[Config] No .ygs/tracker.yml — using auto-detect"
```

Read `~/.claude/skills/you-got-skills/skills/shared/tracker.md`:
- Auto-detect or read tracker type (github | jira)
- Load project, sprint/milestone, team, capacity from `.ygs/tracker.yml`
- **Run the credential verify gate** (tracker.md "Credential verify gate" section)
- If `.ygs/tracker.yml` is absent, infer from current repo; ask user to confirm team list

Read `~/.claude/skills/you-got-skills/skills/shared/slack.md` if Slack signals needed:
- Run the Slack token check; skip gracefully if `SLACK_BOT_TOKEN` unset
