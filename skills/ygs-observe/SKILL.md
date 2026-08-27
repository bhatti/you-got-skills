---
name: ygs-observe
description: Production instrumentation design — structured logging, RED metrics, OpenTelemetry tracing, symptom-based alerting. Use when adding a new feature, service, or critical path that needs to be diagnosable in production.
argument-hint: "[feature or service to instrument]"
---

# Observe

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — instrumentation that ships blind is not instrumentation.

**Instrumentation without a question is noise.** Start with the on-call questions, then pick the signal that answers each one.

For the pre-launch observability gate, read `references/observability-checklist.md`.

## When NOT to use

- Auditing existing code for observability gaps — use `/ygs-sre-review` (it will reference this skill if gaps are found)
- Simple scripts, CLIs, or batch jobs with no production SLA — lightweight logging is sufficient

## Step 1: Define on-call questions first

Before writing a single line of telemetry, write down 2-4 questions that on-call would need to answer during an incident involving this feature:

```
Q1: Is this feature working right now?
Q2: How many users are affected when it fails?
Q3: Where is time being spent?
Q4: What's the error rate trend over the last hour?
```

Every piece of telemetry must answer at least one of these questions. If you can't map a metric/log/trace to a question, don't add it.

## Step 2: Pick the right signal type

| Signal | Answers | When to use |
|--------|---------|-------------|
| **Structured log** | What happened, with what inputs, and what was the outcome | One-time events: requests, errors, state transitions |
| **Metric** | How often, how fast, how many over time | Aggregatable patterns: rate, error %, latency distribution |
| **Trace** | Where did the time go across a request's full path | Latency breakdown, cross-service debugging |

Use all three for production-critical features. Use logs + metrics for most features. Use metrics alone for high-volume hot paths where log volume would be prohibitive.

## Step 3: Structured logging rules

Logs are for events, not prose. Each log line should be parseable.

**Rules:**
- Log **events**, not sentences: `user.login.succeeded` not `"User John logged in successfully"`
- Use **stable event names** + structured fields: `event="request.complete" method="POST" path="/api/orders" status=200 latency_ms=42`
- **Log levels** (tie each to an on-call action):
  - `error` — something failed and on-call may need to act now
  - `warn` — degraded state that may escalate; investigate when convenient
  - `info` — significant business event (request completed, job finished)
  - `debug` — detailed diagnostic; disabled in production by default
- **Correlation IDs are mandatory** for any request-handling code: propagate `trace_id` / `request_id` through all log lines in a request's path
- **Never log PII or secrets**: no passwords, tokens, SSNs, email addresses, credit card numbers

## Step 4: RED metrics for services

For any service handling requests, instrument the RED metrics:

| Metric | Description | Alert on |
|--------|------------|----------|
| **Rate** | Requests per second (or per minute) | Sudden drop (possible traffic loss or crash) |
| **Errors** | Error rate as % of total requests | Rising above baseline threshold |
| **Duration** | Latency distribution (p50, p95, p99) | p99 rising above SLO |

**Cardinality discipline** — cardinality explosions kill metric systems:
- ✅ Labels with bounded values: `method`, `status_code`, `region`, `service`
- ❌ Labels with unbounded values: `user_id`, `request_id`, `order_id`, URLs with IDs in them

Use percentiles (p95, p99), not averages — averages hide the tail where users actually suffer.

For resource-bound components (workers, queues, caches), also instrument USE: Utilization, Saturation, Errors.

## Step 5: Distributed tracing

For multi-service or async features, add OpenTelemetry tracing:

- **One span per meaningful unit of work**: one HTTP handler, one DB query batch, one external API call
- **Propagate context across async boundaries**: message queues, background jobs, async tasks must carry the trace context
- **Add span attributes for diagnosis**: `user.id`, `order.id`, `feature.flag`, operation name — bounded cardinality only
- **Sampling strategy**: 100% in staging, 1-10% in production (or head-based sampling for errors)

Prefer the OpenTelemetry SDK for vendor neutrality — don't tie instrumentation to a specific backend.

## Step 6: Symptom-based alerting

Alert on what users feel, not on what machines do.

**Alert design rules:**
- **Actionable**: on-call must be able to do something about it. No alerts for conditions with no response.
- **Runbook linked**: every alert must have a runbook link in the alert description
- **Justified threshold**: document why the threshold is what it is (SLO target, historical baseline, error budget)
- **Two severities only**: `page` (wake me up now, user-visible impact) vs. `ticket` (investigate during business hours)

**Symptom-first examples:**
- ✅ "Error rate > 5% for 5 minutes" — users are experiencing failures
- ✅ "p99 latency > 2s for 10 minutes" — users are experiencing slowness
- ❌ "CPU > 80%" — not a symptom; users may be fine; ticket at most
- ❌ "Memory usage growing" — not actionable without a threshold and runbook

## Step 7: Verify the telemetry before shipping

Telemetry that hasn't been tested doesn't work. Before marking this done:

- [ ] Force an error condition in staging — verify the error log appears with correct fields
- [ ] Send test traffic — verify RED metrics appear in the dashboard
- [ ] Follow a test request in the tracing UI — verify spans connect end-to-end
- [ ] Test-fire each new alert rule in staging — verify it fires and the runbook link is correct
- [ ] Confirm correlation IDs propagate through all log lines for a single request

Read `references/observability-checklist.md` for the full pre-launch gate.

Report **DONE** or **DONE_WITH_CONCERNS**.

---

## Common Rationalizations

| Rationalization | Reality |
|----------------|---------|
| "We'll add observability after launch when we know what to monitor" | You won't know what to monitor until an incident teaches you. The first incident is the most expensive classroom. |
| "Structured logging is overhead — we'll just use print statements" | Unstructured logs aren't queryable. You'll grep them manually at 2am and miss the signal. |
| "We're too early-stage for tracing" | Distributed tracing is hardest to retrofit. The right time is the first time you build the feature. |
| "Our p50 latency is fine" | p50 means half your users. The other half are experiencing p99. Use percentiles. |
| "I tested it manually and it looked right" | Manual tests don't verify that metrics arrive at the backend, alerts fire correctly, or correlation IDs propagate. Run Step 7. |
