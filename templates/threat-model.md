# Threat Model: [System/Feature Name]

**Date:** YYYY-MM-DD
**Author:** [Name]
**Status:** Draft | Review | Approved
**Design Doc:** [link to source TRD or design doc]

---

## 1. Introduction

### Purpose
<!-- Why this threat model exists and what security decisions it informs -->

### Background
<!-- Brief system description and business context -->

### Scope

**In scope:**
- 

**Out of scope:**
- 

### Key Use Cases
<!-- Primary user/system flows being analyzed -->

| # | Use Case | Actors | Data Flow |
|---|----------|--------|-----------|
| 1 | | | |

### Security Tenets

| Tenet | Description | Priority |
|-------|-------------|----------|
| Confidentiality | | |
| Integrity | | |
| Availability | | |
| Non-repudiation | | |
| Least privilege | | |

---

## 2. STRIDE Quick Reference

| Category | Threat | Property Violated | Example |
|----------|--------|-------------------|---------|
| **S**poofing | Impersonating a user or system | Authentication | Stolen credentials, forged tokens |
| **T**ampering | Modifying data or code | Integrity | SQL injection, parameter manipulation |
| **R**epudiation | Denying actions performed | Non-repudiation | Missing audit logs, unsigned transactions |
| **I**nformation Disclosure | Exposing data to unauthorized parties | Confidentiality | Verbose errors, unencrypted storage |
| **D**enial of Service | Disrupting availability | Availability | Resource exhaustion, amplification attacks |
| **E**levation of Privilege | Gaining unauthorized access | Authorization | IDOR, privilege escalation, confused deputy |

---

## 3. System Architecture

### High-Level Design
<!-- Architecture diagram description or reference -->

### Key Components

| Component | Responsibility | Trust Level | Data Handled |
|-----------|---------------|-------------|--------------|
| | | | |

### Trust Boundaries
<!-- Where trust transitions occur (e.g., public internet → load balancer → internal service → database) -->

1. 
2. 

### API Surface

| Endpoint/Interface | Auth Required | Input Sources | Sensitivity |
|--------------------|---------------|---------------|-------------|
| | | | |

---

## 4. Assets & Data Classification

| Asset | Classification | Storage | Encryption | Retention |
|-------|---------------|---------|------------|-----------|
| | PII / Sensitive / Internal / Public | | At-rest / In-transit | |

---

## 5. Threat Actors

| Actor | Description | Assumed Capability | Motivation |
|-------|-------------|-------------------|------------|
| External attacker | | | |
| Malicious insider | | | |
| Compromised dependency | | | |

---

## 6. Assumptions

| ID | Assumption | Validation Method |
|----|-----------|-------------------|
| A1 | | |

---

## 7. Threat Analysis

<!-- Group threats by subsystem or trust boundary. One table per group. -->

### [Subsystem/Boundary 1]

| ID | Priority | Threat | STRIDE | Affected Assets | Mitigations | Status | Notes |
|----|----------|--------|--------|-----------------|-------------|--------|-------|
| T1 | Critical/High/Medium/Low | | S/T/R/I/D/E | | | Open/Mitigated/Accepted | |

### [Subsystem/Boundary 2]

| ID | Priority | Threat | STRIDE | Affected Assets | Mitigations | Status | Notes |
|----|----------|--------|--------|-----------------|-------------|--------|-------|
| | | | | | | | |

---

## 8. Security Anti-Pattern Review

| Anti-Pattern | Present? | Location | Remediation |
|--------------|----------|----------|-------------|
| Hardcoded secrets | | | |
| Overly broad permissions | | | |
| Missing input validation | | | |
| Plaintext sensitive data | | | |
| Missing rate limiting | | | |
| Verbose error messages | | | |
| Non-constant-time comparisons | | | |
| Predictable identifiers | | | |

---

## 9. Mitigations

| ID | Mitigation | Threats Addressed | Implementation | Status |
|----|-----------|-------------------|----------------|--------|
| M1 | | T1, T2 | | Planned/In Progress/Complete |

---

## 10. Security Tests

| Test ID | Mitigation | Description | Type | Status |
|---------|-----------|-------------|------|--------|
| ST1 | M1 | | Unit/Integration/Pen-test/Automated | |

---

## 11. Compliance & Privacy

### Regulatory Applicability

| Regulation | Applicable? | Requirements | Status |
|-----------|-------------|--------------|--------|
| GDPR | | | |
| SOC 2 | | | |
| HIPAA | | | |
| PCI-DSS | | | |

### Data Inventory (PII/Sensitive)

| Data Element | Source | Purpose | Retention | Deletion Path |
|-------------|--------|---------|-----------|---------------|
| | | | | |

---

## 12. Open Items & Risk Acceptance

| ID | Item | Risk Level | Owner | Decision | Date |
|----|------|-----------|-------|----------|------|
| O1 | | Critical/High/Medium/Low | | Accept/Mitigate/Transfer | |

---

## 13. Appendix

### Glossary
<!-- Domain-specific terms used in this document -->

### References
<!-- Links to design docs, standards, prior threat models -->

### Document History

| Date | Author | Change |
|------|--------|--------|
| | | Initial draft |
