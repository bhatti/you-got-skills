# Tracker Abstraction

DRY query patterns for GitHub (`gh`) and JIRA/Bitbucket (`acli`).
All team intelligence skills read this before making tracker calls.

---

## Environment variables

```bash
# GitHub — set by `gh auth login`

# JIRA (Basic Auth: base64 of email:token)
# JIRA_BASE_URL=https://yourorg.atlassian.net
# JIRA_EMAIL=you@company.com
# JIRA_API_TOKEN=ATATT3x...   (from id.atlassian.com/manage-profile/security/api-tokens)
# JIRA_AUTH=$(echo -n "$JIRA_EMAIL:$JIRA_API_TOKEN" | base64)

# Slack — see shared/slack.md
# SLACK_BOT_TOKEN=xoxb-...
```

## Credential verify gate

**Call this at the start of every skill that uses tracker data.** Empty results from a
misconfigured token are indistinguishable from real "no activity" — a silent false signal.

```bash
# JIRA
if [ -n "$JIRA_AUTH" ]; then
  curl -sf "$JIRA_BASE_URL/rest/api/3/myself" \
    -H "Authorization: Basic $JIRA_AUTH" > /dev/null \
    || { echo "[Tracker] ERROR — JIRA credentials invalid. Check JIRA_EMAIL/JIRA_API_TOKEN."; exit 1; }
fi

# GitHub (gh auth status exits non-zero if not logged in)
if [ "$TRACKER" = "github" ]; then
  gh auth status >/dev/null 2>&1 \
    || { echo "[Tracker] ERROR — not logged in. Run: gh auth login"; exit 1; }
fi
```

## Configuration

Read `.ygs/tracker.yml` at project root. If absent, auto-detect:

```bash
if [ -f .ygs/tracker.yml ]; then
  TRACKER=$(grep '^tracker:' .ygs/tracker.yml | awk '{print $2}')
elif acli -p jira jira user me >/dev/null 2>&1; then
  TRACKER=jira
else
  TRACKER=github
fi
```

### `.ygs/tracker.yml` schema (canonical — example file references this)

```yaml
tracker: github              # github | jira

github:
  org: myorg                 # defaults to current repo owner
  repo: myrepo               # defaults to current repo
  sprint_milestone: Sprint 5 # milestone name used as sprint proxy

jira:
  base_url: https://myorg.atlassian.net  # also set JIRA_BASE_URL env var
  project: PROJ              # project key (e.g. CRIBL, ENG)
  board_id: 42               # numeric sprint board ID (from board URL)
  # epic: PROJ-100           # optional: scope to one epic

bitbucket:
  workspace: myworkspace
  repo: myrepo

slack:
  standup_channel: "#standup"
  team_channel: "#engineering"   # optional: broader signal scan

team:                        # tracker usernames — exact match to assignee field
  - alice
  - bob

capacity:
  hours_per_person_per_day: 6
  sprint_days: 10
  meeting_overhead_pct: 20

# Note: commit tracker.yml for shared team config; gitignore it if org/team names are sensitive
```

---

## Semantic queries

### Issues updated recently (standup signal)

```bash
# GitHub
gh issue list --state open \
  --json number,title,assignees,labels,updatedAt,milestone,body \
  --jq '[.[] | select(.updatedAt > (now - 86400 | todate))]'

# JIRA (acli preferred; curl fallback)
acli -p jira jira issue search \
  --jql "project = PROJ AND updated >= -1d AND sprint in openSprints() ORDER BY assignee" \
  2>/dev/null || \
curl -sf "$JIRA_BASE_URL/rest/api/3/search" \
  -H "Authorization: Basic $JIRA_AUTH" \
  -G --data-urlencode \
  "jql=project=PROJ AND updated>=-1d AND sprint in openSprints() ORDER BY assignee" \
  | jq '.issues[]'
```

### Issues in current sprint

```bash
# GitHub — sprint = milestone
gh issue list --milestone "Sprint N" --state open \
  --json number,title,assignees,labels,updatedAt

# JIRA
acli -p jira jira issue search \
  --jql "project = PROJ AND sprint in openSprints() ORDER BY priority DESC"
```

### Issues by assignee

```bash
# GitHub
gh issue list --assignee USERNAME --state open --json number,title,labels,updatedAt

# JIRA
acli -p jira jira issue search \
  --jql "project = PROJ AND assignee = USERNAME AND sprint in openSprints()"
```

### Issues by epic

```bash
# GitHub — epics as labels
gh issue list --label "epic:auth" --state open --json number,title,assignees,updatedAt

# JIRA (JIRA Cloud: "Epic Link"; Next-gen: parent)
acli -p jira jira issue search \
  --jql 'project = PROJ AND "Epic Link" = PROJ-123 AND sprint in openSprints()'
```

### Stale issues (no activity > N days)

```bash
# GitHub — embed threshold in the jq expression directly
DAYS=3
gh issue list --state open --json number,title,assignees,updatedAt \
  --jq "[.[] | select(.updatedAt < (now - ($DAYS * 86400) | todate))]"

# JIRA
acli -p jira jira issue search \
  --jql "project = PROJ AND updated <= -${DAYS}d AND sprint in openSprints() AND status != Done"
```

### Blocked issues

```bash
# GitHub — blocked label
gh issue list --label blocked --state open --json number,title,assignees,updatedAt,body

# JIRA — status or issue link (standard JQL only — no ScriptRunner required)
acli -p jira jira issue search \
  --jql 'project = PROJ AND (status = "Blocked" OR labels = "blocked")'
# To find issues with "is blocked by" links, use the REST API:
curl -sf "$JIRA_BASE_URL/rest/api/3/search" \
  -H "Authorization: Basic $JIRA_AUTH" \
  -G --data-urlencode \
  'jql=project = PROJ AND sprint in openSprints() AND issueLinks is not EMPTY' \
  | jq '[.issues[] | select(.fields.issuelinks[]?.type.inward == "is blocked by")]'
```

### Sprint end date and capacity (for risk-scan / sprint-plan)

```bash
# JIRA — fetch active sprint to get end date
SPRINT_ID=$(curl -sf "$JIRA_BASE_URL/rest/agile/1.0/board/$BOARD_ID/sprint?state=active" \
  -H "Authorization: Basic $JIRA_AUTH" \
  | jq -r '.values[0].id')
SPRINT_END=$(curl -sf "$JIRA_BASE_URL/rest/agile/1.0/sprint/$SPRINT_ID" \
  -H "Authorization: Basic $JIRA_AUTH" \
  | jq -r '.endDate')  # ISO8601

# GitHub — milestone due date as sprint end proxy
gh api repos/ORG/REPO/milestones --jq '[.[] | select(.title == "Sprint N")][0].due_on'
```

### Open PRs

```bash
# GitHub
gh pr list --state open \
  --json number,title,author,reviewers,createdAt,updatedAt,isDraft,labels

# Bitbucket
acli -p bitbucket bitbucket pr list \
  --workspace WORKSPACE --repo REPO --state OPEN
```

### Stale PRs (no review activity > N days)

```bash
# GitHub — embed threshold directly in jq expression
DAYS=2
gh pr list --state open --json number,title,author,reviewers,updatedAt \
  --jq "[.[] | select(.updatedAt < (now - ($DAYS * 86400) | todate))]"

# Bitbucket
CUTOFF=$(date -d "${DAYS} days ago" --iso-8601=seconds 2>/dev/null \
  || date -v-${DAYS}d +%Y-%m-%dT%H:%M:%S)
acli -p bitbucket bitbucket pr list \
  --workspace WORKSPACE --repo REPO --state OPEN \
  | jq --arg cutoff "$CUTOFF" '[.[] | select(.updatedOn < $cutoff)]'
```

### Recent velocity (for sprint planning)

```bash
# GitHub — count closed issues per last 2 milestones
gh issue list --state closed --milestone "Sprint N-1" --json closedAt | jq length
gh issue list --state closed --milestone "Sprint N-2" --json closedAt | jq length

# JIRA
acli -p jira jira issue search \
  --jql "project = PROJ AND sprint in closedSprints() AND status = Done ORDER BY sprint DESC" \
  --max-results 100
```

### PR review detail

```bash
# GitHub
gh pr view NUMBER --json comments,reviews,updatedAt

# Bitbucket
acli -p bitbucket bitbucket pr get --workspace WORKSPACE --repo REPO --id NUMBER
```

### Resolve JIRA sprint ID by name (for sprint assignment)

```bash
# Required before assigning issues to a sprint — JIRA needs numeric ID
curl -sf "$JIRA_BASE_URL/rest/agile/1.0/board/$BOARD_ID/sprint?state=active,future" \
  -H "Authorization: Basic $JIRA_AUTH" \
  | jq -r '.values[] | select(.name == "Sprint N") | .id'
```

---

## Normalize output

After gathering, normalize to a common shape before synthesis:

```json
{
  "issues": [
    {
      "id": "PROJ-123 | #42",
      "title": "...",
      "assignee": "alice",
      "status": "In Progress | open",
      "updated_at": "ISO8601",
      "days_stale": 0,
      "blocked": false,
      "sprint": "Sprint 5",
      "epic": "epic-name | null",
      "url": "https://..."
    }
  ],
  "prs": [
    {
      "id": "#456 | PR-78",
      "title": "...",
      "author": "bob",
      "reviewers": ["charlie"],
      "created_at": "ISO8601",
      "days_open": 3,
      "has_review_activity": false,
      "url": "https://..."
    }
  ],
  "sprint": {
    "end_date": "ISO8601",
    "days_remaining": 4,
    "total_issues": 14,
    "done": 8,
    "in_progress": 3,
    "not_started": 3
  }
}
```
