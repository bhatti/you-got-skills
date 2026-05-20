---
name: ygs-sre-review
description: SRE/operational review — failure modes, observability, stateful components, rollback safety, on-call impact. Use for production-bound changes.
---

# SRE Review

For detailed checklists, read:
- `references/production-readiness.md` — Deployment, resilience, observability, DR
- `references/testing-reliability.md` — Test pyramid, chaos, property-based, caching scenarios

## Step 1: Get scope

```bash
git diff main...HEAD --name-only 2>/dev/null || git diff master...HEAD --name-only 2>/dev/null
```

Identify changes to: infrastructure, configuration, stateful services, deployment configs, database schemas, background jobs, queues, caches.

## Step 2: Failure mode analysis

For each changed component:

1. **What happens when it fails?** — Crash, hang, corrupt data, silent degradation?
2. **Blast radius** — Does failure cascade? What other services are affected?
3. **Recovery path** — Auto-recovers? Requires manual intervention? How long?
4. **Data durability** — Can data be lost during failure? Is it recoverable?
5. **Partial failure** — What if it half-succeeds? Inconsistent state possible?

## Step 3: Observability

1. **Logging** — Are failures logged with enough context to diagnose? Structured key-value pairs?
2. **Metrics** — Are key operations instrumented (latency, error rate, throughput)?
3. **Alerting** — Would on-call be paged for failures? Are thresholds appropriate?
4. **Tracing** — Can a request be traced end-to-end through this change?
5. **Health checks** — Updated to reflect new dependencies?

## Step 4: Operational risk

1. **Stateful changes** — New persistent state? Migration path? Backwards-compatible schema?
2. **Resource consumption** — Memory growth? Connection pool exhaustion? Disk usage?
3. **Concurrency** — Race conditions under load? Thundering herd? Hot keys?
4. **Dependencies** — New external dependencies? Timeout/retry/circuit-breaker configured?
5. **Capacity** — Can this handle 10x current load? Where does it break first?

## Step 5: Deployment safety

1. **Rollback** — Can this be rolled back without data loss? Does rollback require migration?
2. **Feature flags** — Should this be behind a flag for gradual rollout?
3. **Backwards compatibility** — Can old and new versions coexist during deploy?
4. **Database migrations** — Online-safe? Locks tables? Backwards-compatible?
5. **Configuration changes** — Can old config format still be read by new code?

## Step 6: On-call impact

- Does this change what on-call needs to know?
- New runbook entries needed?
- New alerts that will fire?
- Changed behavior that existing dashboards won't show?

## Step 7: Report

For each finding:
- **Severity:** Critical (blocks deploy) | High | Medium | Low
- **Category:** Failure mode | Observability | Capacity | Deployment
- **Location** and description
- **Recommended mitigation**

Report **DONE** or **DONE_WITH_CONCERNS**.
