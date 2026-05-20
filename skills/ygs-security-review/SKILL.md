---
name: ygs-security-review
description: Security audit and red-team review. Checks OWASP top 10, auth, secrets, injection, supply chain, and adversarial attack paths. Best with reasoning model (Opus-class).
---

# Security Review

For detailed checklist, read `references/microservice-security.md`.

## Step 0: Read references

Read `~/.claude/skills/you-got-skills/references/microservice-security.md` for detailed checklist.
Read `~/.claude/skills/you-got-skills/templates/threat-model.md` for output structure.

## Step 1: Get scope

```bash
git diff main...HEAD --name-only 2>/dev/null || git diff master...HEAD --name-only 2>/dev/null
```

Read full file contents (not just diffs) for security-relevant files.

## Step 2: Systematic audit

### Injection vectors
- SQL injection (raw queries, string interpolation in queries)
- Command/shell injection (exec, system calls with user input)
- Template injection (server-side template rendering with user data)
- XSS (user content rendered without escaping)
- SSRF (user-controlled URLs in server-side requests)
- Path traversal (user input in file paths without sanitization)
- Header injection (user input in HTTP headers)

### Authentication & Authorization
- Missing auth checks on endpoints
- Broken access control (IDOR, privilege escalation)
- Weak session management (predictable tokens, missing expiry)
- Credential handling (plaintext storage, weak hashing like MD5/SHA1)

### Data exposure
- Sensitive data in logs, error messages, or API responses
- Hardcoded secrets (keys, tokens, passwords in code or config)
- Credentials in URLs or query parameters
- Debug mode enabled in production configs

### Cryptography
- Weak algorithms (MD5, SHA1 for security purposes)
- Predictable randomness (Math.random for tokens)
- Non-constant-time comparisons for secrets
- Missing encryption for sensitive data at rest/transit

### Supply chain
- Known vulnerable dependencies (check lock files)
- Unpinned dependencies that could be hijacked
- Deserialization of untrusted data (pickle, YAML.load, eval)

### Configuration
- Permissive CORS
- Missing security headers (CSP, HSTS, X-Frame-Options)
- Exposed debug endpoints or admin panels
- Overly permissive IAM/RBAC

## Step 3: Red-team perspective

For each exposed endpoint or user-facing flow, think like an attacker:
- What happens at 10x expected load?
- What if the database is slow? External service returns garbage?
- What happens on double-click within 100ms?
- What if frontend validation is bypassed entirely?
- What's the blast radius if this is compromised?
- Is there defense in depth or single point of failure?

### Silent failure exploitation
- Exception swallowing that hides attacks
- Partial completion (3/5 items processed then crash — inconsistent state)
- Jobs failing silently with no alerting

## Step 4: Report

For each finding:
- **Severity:** Critical | High | Medium | Low
- **Location:** file:line
- **Attack scenario:** How an attacker exploits this
- **Recommended fix**

Overall risk assessment and priority order for fixes.

## Step 5: Threat model (if requested)

If the user asks for a formal threat model, write one at `docs/threat-models/YYYY-MM-DD-<slug>.md` using the template structure from `~/.claude/skills/you-got-skills/templates/threat-model.md`.

Report **DONE** or **DONE_WITH_CONCERNS**.
