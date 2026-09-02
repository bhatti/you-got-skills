# Specialist: Security

Review scope: can this code be exploited, abused, or used to expose data it should not?

## Injection vectors

- **SQL injection** — user input in query strings without parameterized queries
- **Command injection** — user input passed to shell execution
- **Template injection** — user input rendered in server-side templates without escaping
- **XSS** — user input rendered in HTML without encoding; `innerHTML`, `dangerouslySetInnerHTML`
- **SSRF** — user-controlled URL fetched server-side without allowlist validation
- **Path traversal** — user input used in file paths without canonicalization

## Authentication & authorization

- Missing authentication check on a new endpoint or route
- Authorization check on wrong object (checking the parent but not the resource)
- Privilege escalation: can a lower-privilege user trigger a higher-privilege action?
- Session fixation or token reuse across contexts

## Secrets & credentials

- Hardcoded secrets, tokens, passwords, or API keys in code or config
- Secrets logged at any log level
- Credentials stored in plaintext or weakly encoded (base64 is not encryption)
- Environment variable names that suggest secrets (check they are not defaulting to insecure values)

## Cryptography

- Weak algorithms: MD5/SHA1 for security purposes, DES/3DES, RC4
- Predictable randomness: `Math.random()`, `rand()`, seeded with time for security tokens
- Timing attacks: non-constant-time comparison of secrets or tokens

## Supply chain

Reference `~/.claude/skills/you-got-skills/skills/shared/dep-audit.md` for dependency vulnerability scanning.

- New dependency added without pinned version
- Dependency with known CVE at the version pinned
- Dependency with an unusual maintainer or recent ownership change

## Severity guidance

| Severity | Example |
|----------|---------|
| MUST | Injection vector, missing auth check, hardcoded secret |
| MUST | Weak crypto on sensitive data, timing attack on token comparison |
| SHOULD | Unpinned dependency, missing input sanitization on low-risk input |
| MAY | Logging verbosity that could leak non-sensitive internal state |
