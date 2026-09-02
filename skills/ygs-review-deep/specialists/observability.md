# Specialist: Observability

Review scope: can an on-call engineer understand what this code is doing in production without a debugger?

For the full instrumentation design guide, read `~/.claude/skills/you-got-skills/skills/ygs-observe/SKILL.md`.

## Structured logging

- New code paths that can fail have at least one log at an appropriate level
- Log events are structured (fields, not interpolated strings): `log.Error("payment_failed", "order_id", id, "error", err)` not `log.Errorf("payment %s failed: %s", id, err)`
- Correlation IDs propagated through log fields (request ID, trace ID, tenant ID)
- No PII or secrets logged at any level
- Debug/trace logs gated by log level — no bare `eprintln!`, `fmt.Println`, `console.log` in production paths

## Log level discipline

| Level | When to use |
|-------|------------|
| ERROR | A human should investigate this — page-worthy |
| WARN | Unexpected but recoverable; watch for frequency |
| INFO | Significant state transition (request received, job started/completed) |
| DEBUG | Diagnostic detail; must be gated by level check |

## Metrics (RED)

- **Rate**: is a counter added for new request/operation types?
- **Errors**: is an error counter incremented on failure paths?
- **Duration**: is a histogram/timer added for new latency-sensitive paths?
- No high-cardinality label values (user IDs, request IDs in metric labels → memory explosion)

## Distributed tracing

- New async or cross-service operations have spans
- Trace context propagated across async boundaries (thread pools, message queues, HTTP calls)
- Span names are stable and low-cardinality (no user IDs in span names)

## Alertability

- Would a failure in this new code path be detectable via existing alerts?
- If not, is the code path important enough to warrant a new alert?

## Severity guidance

| Severity | Example |
|----------|---------|
| MUST | Ungated debug logs or PII in log output |
| MUST | Critical failure path with no log and no metric |
| SHOULD | New latency-sensitive path with no duration metric |
| SHOULD | High-cardinality metric label added |
| MAY | Missing DEBUG log that would aid future diagnosis |
