# Microservice Security Checklist

## Foundation (CIA Triad)
- **Confidentiality** — Encryption, access controls, strong authentication
- **Integrity** — Cryptographic hashes, digital signatures, data validation
- **Availability** — Redundancy, failover, regular maintenance

## Authentication & Credentials
- Multi-factor authentication (MFA) enforced
- Strong password policies and secure credential storage
- Credential rotation automated
- No hardcoded credentials in code or config
- Temporary/conditional access for privileged operations

## Authorization & Access Control
- Least privilege principle applied to all services
- Zero trust model — verify every access attempt regardless of source
- Separation of duties — no single entity controls entire flow
- Multi-tenant isolation — robust boundaries between tenants
- RBAC/ABAC properly configured and audited

## Data Protection
- Data classified by sensitivity level
- Encryption at rest and in transit (TLS with strong ciphers)
- PII minimized, anonymized where possible
- Data masking in non-production environments
- Secure key management with rotation
- No PII in logs, error messages, or URLs

## Input & Communication
- All input validated at trust boundaries
- Deserialization of untrusted data prevented
- Template injection mitigated (SSTI)
- SQL/command/LDAP injection prevented
- XSS, CSRF, SSRF protections in place
- Content Security Policy headers configured

## Supply Chain & Dependencies
- Dependency inventory maintained
- Known vulnerabilities scanned (Snyk, Dependabot, etc.)
- Code signing for all deployed artifacts
- Package integrity verified (checksums, signatures)
- No unverified third-party packages

## Infrastructure
- Secure defaults (no open-by-default configurations)
- Filesystem permissions hardened
- Memory protection (ASLR/KASLR enabled)
- Regional/partition isolation for blast radius control
- Network segmentation between services
- Certificate management with rotation and revocation

## Monitoring & Response
- Comprehensive secure logging (no sensitive data in logs)
- Real-time alerting for security events
- Audit trails for all administrative actions
- Incident response procedures documented
- Backup and disaster recovery tested

## Frameworks to Reference
- OWASP Top 10
- CWE Top 25
- STRIDE threat modeling
- NIST Cybersecurity Framework
