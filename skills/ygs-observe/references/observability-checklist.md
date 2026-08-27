# Observability Pre-Launch Checklist

Use this as a gate before shipping any production-bound feature. Also referenced by `ygs-sre-review` Step 3 when instrumentation is absent or thin.

## Logging

- [ ] All errors logged at `error` level with: event name, error message, request/correlation ID, relevant context fields
- [ ] No PII or secrets in any log line
- [ ] Log levels are correctly calibrated: `debug` disabled in production, `info` only for significant events
- [ ] Structured key-value format (not prose sentences)
- [ ] Correlation ID propagated through all log lines in a request path

## Metrics

- [ ] RED metrics instrumented: Rate (requests/sec), Errors (error rate %), Duration (p50/p95/p99)
- [ ] No unbounded label values (no user IDs, request IDs, or URL path parameters as label dimensions)
- [ ] Metrics visible in the dashboard for the feature's critical path
- [ ] Baselines documented: what is "normal" rate, error %, and latency for this feature?

## Tracing

- [ ] Trace context propagated across all async boundaries (queues, background jobs, external calls)
- [ ] Span attributes include business context (operation name, relevant entity IDs — bounded cardinality)
- [ ] A test request can be traced end-to-end in the tracing UI

## Alerting

- [ ] At least one alert covers the feature's error rate
- [ ] At least one alert covers latency SLO if feature is user-facing
- [ ] Each alert has: actionable condition, runbook link, justified threshold, correct severity (page vs. ticket)
- [ ] Alerts test-fired in staging and confirmed to fire correctly

## On-call Readiness

- [ ] On-call questions defined (2-4 questions per feature — see ygs-observe Step 1)
- [ ] Runbook entry written for new failure modes introduced by this feature
- [ ] Dashboard updated or created to show the new feature's health
