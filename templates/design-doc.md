# Design Doc: [Feature/System Name]

**Date:** YYYY-MM-DD
**Author:** [Name]
**Status:** Draft | Review | Approved
**PRD:** [link to source PRD]

---

## 1. Executive Summary

<!-- 2-3 sentences: what are we building, why now, expected impact -->

---

## 2. Background & Problem Statement

### Current State
<!-- How things work today. Diagrams if helpful. -->

### Problem
<!-- What's broken, slow, risky, or missing. Quantify where possible (latency, error rate, cost). -->

### Why Now
<!-- What triggered this work? Business event, scaling threshold, tech debt tipping point? -->

---

## 3. Proposal & Stakeholders

### Proposed Solution
<!-- High-level description of what we're building. One paragraph, then detail below. -->

### Stakeholders

| Role | Who | Concerns |
|------|-----|----------|
| Owner | | |
| Reviewer | | |
| Downstream consumer | | |
| On-call/SRE | | |

### Supporting Data
<!-- Metrics, user research, benchmarks, load projections that justify the approach -->

---

## 4. Architecture

### System Diagram
<!-- Describe components, interactions, data flow. Reference a diagram file if available. -->

### Components

| Component | Responsibility | Owner | New/Modified |
|-----------|---------------|-------|--------------|
| | | | |

### Data Model
<!-- Key entities, relationships, schema changes. Mark BREAKING changes. -->

### API Design
<!-- Endpoints or interfaces. Request/response shapes. Versioning strategy. -->

### Data Flow
<!-- End-to-end request path. Include async flows, queues, caches. -->

### Failure Paths
<!-- What happens when each component fails? Cascading failure risks? Recovery? -->

### Dependencies
<!-- External services, libraries, infrastructure. Timeout/retry/circuit-breaker for each. -->

---

## 5. Alternatives Considered

| Option | Pros | Cons | Why Not |
|--------|------|------|---------|
| Do nothing (current state) | | | |
| Incremental improvement | | | |
| Full redesign | | | |
| [Proposed approach] | | | Selected |

---

## 6. Functional Requirements

<!-- Derived from PRD. Trace back to source requirement. -->

| ID | Requirement | Priority | PRD Source | Verification |
|----|-------------|----------|------------|--------------|
| FR1 | | P0/P1/P2 | | |

### Acceptance Scenarios

<!-- Given/When/Then for key flows -->

```
Given [precondition]
When [action]
Then [expected outcome]
```

---

## 7. Non-Functional Requirements

### Performance & Scale
<!-- Latency targets, throughput, concurrency. At current scale AND projected growth. -->

| Metric | Target | Current | At 10x |
|--------|--------|---------|--------|
| | | | |

### Availability & Fault Tolerance
<!-- SLA, redundancy, graceful degradation, circuit breakers -->

### Security & Privacy
<!-- Auth model, encryption, trust boundaries, data classification. Reference threat model if exists. -->

### Testing Strategy
<!-- Test pyramid: unit/integration/e2e. Key test seams. Chaos/load testing plans. -->

### Observability & Operations
<!-- Metrics, logs, traces, alerts, dashboards, runbooks needed -->

### Cost
<!-- Infrastructure cost estimate. Cost per request/user if applicable. -->

### Compliance
<!-- Regulatory requirements, data retention, audit logging -->

---

## 8. Rollout & Future Plans

### Migration Plan
<!-- How to deploy safely: feature flags, phased rollout, data migration, rollback path -->

### Release Phases

| Phase | Scope | Success Criteria | Timeline |
|-------|-------|-----------------|----------|
| 1 | | | |

### Future Work
<!-- Explicitly deferred scope. Not in this design but planned for later. -->

---

## 9. Decision Log

<!-- Key decisions made during design. Format as lightweight ADRs. -->

### Decision 1: [Title]
- **Context:** 
- **Decision:** 
- **Rationale:** 
- **Consequences:** 

### Decision 2: [Title]
- **Context:** 
- **Decision:** 
- **Rationale:** 
- **Consequences:** 

---

## Open Questions

<!-- Unresolved issues requiring input before implementation -->

| # | Question | Owner | Resolution | Date |
|---|----------|-------|------------|------|
| 1 | | | | |
