# Non-Functional Requirements Checklist

Canonical checklist for verifying NFRs are present and concrete at each stage of the SDLC. Referenced by `ygs-review-prd`, `ygs-review-trd`, `ygs-review-architecture`, and `ygs-review-pr` (SRE domain).

**Principle:** A vague NFR ("fast", "secure", "scalable") cannot be tested, monitored, or validated. An absent NFR is silently out of scope until it becomes an incident.

---

## How to Use

At each stage, check each category below. For each category, the question is the same:
1. Is it **present** in the document/design?
2. Is it **concrete** (a number, a named model, a specific policy)?
3. Is it explicitly **scoped out** with justification if absent?

A "scoped out" statement (`"Performance targets are out of scope for v1 — will be addressed when user load exceeds 100 concurrent users"`) is acceptable. Silence is not.

---

## Categories

### Performance
**What to verify:**
- Response time targets: p50, p95, p99 latency for each key operation
- Throughput: requests per second (RPS), transactions per second, jobs per minute
- Latency budget: how is latency allocated across tiers (client → API → DB)?

**Red flags:** "Should be fast", "must be responsive", "low latency" with no number.
**Good signal:** "p99 < 200ms for the search endpoint at 500 RPS sustained load"

---

### Scalability
**What to verify:**
- Expected max concurrent users at launch; projected growth horizon
- Max data volume the system must handle without degradation
- Where does it break at 10× current load? At 100×?
- Horizontal vs vertical scaling assumptions (which components scale out?)

**Red flags:** "Will scale as needed", "designed for scale" with no numbers.
**Good signal:** "Designed for 10,000 concurrent connections; sharding strategy kicks in above 100M records"

---

### Reliability / SLO
**What to verify:**
- Availability target (e.g., 99.9% = 8.7h downtime/year, 99.99% = 52min/year)
- Error budget: acceptable error rate per rolling window
- RTO (Recovery Time Objective): how long to restore service after failure?
- RPO (Recovery Point Objective): how much data loss is acceptable?
- Retry/backoff strategy for external dependencies

**Red flags:** "Highly available", "fault-tolerant" with no availability figure or recovery targets.
**Good signal:** "99.9% availability SLO; RTO < 15 min; RPO < 1 hour; retries use exponential backoff with max 3 attempts"

---

### Security
**What to verify:**
- Authentication model: who can call this service/API/function?
- Authorization model: what can each authenticated caller do?
- Data classification: what data does this system handle (PII, secrets, confidential, public)?
- Threat model: top 3 threats and their mitigations
- Secrets management: where are credentials stored and rotated?

**Red flags:** "Follows standard security practices", "will be secured", no auth model named.
**Good signal:** "mTLS between services; JWT with 1-hour expiry for user sessions; PII fields encrypted at rest with AES-256; top threats: credential theft (mitigated by short-lived tokens), SSRF (mitigated by egress allowlist)"

---

### Observability
**What to verify:**
- Which operations emit metrics? What are the metric names and units?
- What is traced end-to-end? (request ID propagation, distributed trace spans)
- What are the alerting thresholds and who gets paged?
- Where is the runbook for new failure modes?
- Log levels and sampling strategy for high-volume paths

**Red flags:** "We'll add monitoring later", no named metrics, no alerting thresholds.
**Good signal:** "Request count + error rate + p99 latency exported for each endpoint; distributed trace spans for all DB and external calls; alert fires at error rate > 1% for 5 min; runbook in ops/runbooks/service-name.md"

---

### Data
**What to verify:**
- Retention policy: how long is data kept? When is it purged/archived?
- Backup frequency and recovery procedure
- Schema migration strategy: forward-compatible? Backward-compatible? Two-phase deploy needed?
- Consistency model: eventual vs strong? What are the consistency guarantees to clients?

**Red flags:** No retention policy, no migration plan for schema changes, "we'll figure it out".
**Good signal:** "90-day retention with automated archival to cold storage; daily backups with 30-day retention; migrations are additive-only in v1 (no column drops/renames)"

---

### Compliance
**What to verify:**
- Applicable regulatory requirements (GDPR, HIPAA, SOC 2, PCI-DSS, CCPA)
- Audit trail requirements: what actions must be logged with actor, timestamp, and resource?
- Data residency constraints: where can data be stored and processed?
- Right-to-erasure or right-to-access requirements (GDPR Article 17/15)

**Red flags:** No compliance section when the system processes personal data or financial data.
**Good signal:** "GDPR Article 17 (erasure) implemented via soft-delete + scheduled purge; audit log captures all write operations with user ID and timestamp; data stored in EU-West-1 only"

---

## Stage-Specific Usage

| Stage | What to check |
|-------|--------------|
| PRD review | All 7 categories: at minimum, each must be present, concrete, or explicitly scoped out |
| TRD review | NFR traceability: each PRD NFR must map to a named design decision in the TRD |
| Architecture review | NFR quantitative backing: each category must have a concrete number or scope-out in the doc |
| PR review (SRE domain) | Reliability, Observability, Data: does this PR introduce or regress any of these? |
