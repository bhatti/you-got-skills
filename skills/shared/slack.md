# Slack Integration

Slack signal gathering via Web API. All team intelligence skills reference this.

---

## Prerequisites

**Required bot scopes:** `channels:history`, `channels:read`, `groups:history`, `groups:read`, `users:read`

> `search:read` is a **user-token-only** scope — bot tokens (`xoxb-`) cannot use `search.messages`.
> Use `conversations.history` + text filtering instead (shown below).

**Setup:**
1. api.slack.com/apps → Create New App → From scratch
2. OAuth & Permissions → Bot Token Scopes → add scopes above
3. Request IT approval (most orgs require `#ask-it` ticket)
4. After approval: Install to Workspace → copy Bot User OAuth Token (`xoxb-...`)
5. Invite bot to channels: `/invite @YourBotName` in each channel it needs to read
6. Set `SLACK_BOT_TOKEN=xoxb-...` in environment or secrets manager

**Required env vars:** `SLACK_BOT_TOKEN`

```bash
# Verify token — run this before any skill queries
if [ -z "$SLACK_BOT_TOKEN" ]; then
  echo "[Slack] SKIP — SLACK_BOT_TOKEN not set"; exit 0
fi
curl -s "https://slack.com/api/auth.test" \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  | jq -e '.ok' > /dev/null || { echo "[Slack] ERROR — invalid token"; exit 1; }
```

---

## Queries

### Resolve channel ID (cache per session)

```bash
# conversations.list paginates at 200; use limit+cursor for large workspaces
CHANNEL_NAME="standup"   # override from .ygs/tracker.yml
CHANNEL_ID=$(curl -s "https://slack.com/api/conversations.list?limit=1000" \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  | jq -r --arg name "$CHANNEL_NAME" '.channels[] | select(.name == $name) | .id')
```

### Recent channel messages with blocker signal filtering (last 24h)

```bash
OLDEST=$(date -d '24 hours ago' +%s 2>/dev/null || date -v-24H +%s)
curl -s "https://slack.com/api/conversations.history" \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -d "channel=$CHANNEL_ID&oldest=$OLDEST&limit=200" \
  | jq '[.messages[] | {user: .user, text: .text, ts: .ts}
         | select(.text | test("blocked|stuck|waiting on|help|can.t proceed"; "i"))]'
```

### Post standup brief

```bash
# Use jq to build JSON safely — avoids shell injection on quotes/newlines in brief text
jq -n --arg channel "#standup" --arg text "$BRIEF_TEXT" \
  '{channel: $channel, text: $text, unfurl_links: false}' \
| curl -s "https://slack.com/api/chat.postMessage" \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d @-
```

---

## Signal extraction

Look for these patterns in message text after fetching:

| Pattern | Signal |
|---------|--------|
| `blocked`, `stuck`, `can't proceed` | Blocker |
| `waiting on @person` | Dependency risk |
| `help`, `anyone know` | Question needing answer |
| `PR ready`, `ready for review` | PR needs attention |
| No message from person in last 2+ days | Silence signal |

---

## Graceful degradation

When `SLACK_BOT_TOKEN` is absent or invalid, note the gap and continue:

```
[Slack] Skipped — SLACK_BOT_TOKEN not set (or invalid).
Brief will use tracker signals only (no hallway-conversation layer).
```
