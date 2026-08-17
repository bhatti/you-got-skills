# Risk Classification

Used by `ygs-standup` and `ygs-risk-scan`.

## Severity levels

| Level | Definition |
|-------|-----------|
| **HIGH** | Blocks another person's work, or sprint goal is at risk if unaddressed today |
| **MEDIUM** | No immediate blocker but trajectory is bad — will become HIGH within 2 days |
| **LOW** | Worth noting, not worth meeting time |

## Signal → Risk mapping

| Signal | Default severity | Escalate to HIGH if... |
|--------|-----------------|----------------------|
| Issue stale > 3 days | MEDIUM | It blocks another issue or is on critical path |
| Issue stale > 5 days | HIGH | — |
| PR open > 2 days, no review | MEDIUM | Only one reviewer available and they are unresponsive |
| PR open > 4 days | HIGH | — |
| PR open > 7 days, no activity | HIGH | Suggest: close or assign reviewer with 15-min time estimate |
| Person silent in standup channel > 2 days | MEDIUM | Also no tracker activity |
| Blocked label/status | HIGH | — |
| Dependency chain: A blocks B, A stale | HIGH | — |
| Sprint end < 2 days, issue not started | HIGH | — |
| Sprint end < 2 days, issue in progress | MEDIUM | Assignee has multiple open items |
| P1/P2 issue unassigned | HIGH | Sprint end < 4 days |
| Status mismatch: "In Review" but PR is draft | MEDIUM | Board should reflect reality |
| High WIP: team has > 2× members items in-progress | MEDIUM | Suggests finishing before starting new work |
| PR with open tasks/comments blocking merge | MEDIUM | Vulnerability or OOM fix (urgency keywords in title) |

## What is NOT a risk

- Completed work (closed issues, merged PRs)
- Issues explicitly marked "on hold" or "won't do"
- Work in progress with recent activity (< 1 day)
- Low-priority issues with no dependencies
