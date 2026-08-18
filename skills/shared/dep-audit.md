# Dependency Audit

Polyglot commands for scanning dependencies for known vulnerabilities.
Referenced by security-review and sre-review — do not inline these commands.

## Commands by ecosystem

```bash
# Node.js / npm
npm audit --audit-level=high

# Node.js / yarn
yarn audit --level high

# Node.js / pnpm
pnpm audit --audit-level high

# Rust / Cargo
cargo audit

# Python / pip
pip-audit

# Python / poetry
poetry run pip-audit

# Go
govulncheck ./...

# Ruby / Bundler
bundle audit check --update

# Java / Maven
mvn org.owasp:dependency-check-maven:check

# Java / Gradle
./gradlew dependencyCheckAnalyze

# .NET
dotnet list package --vulnerable
```

## Interpretation

- **Critical / High** — treat as MUST FIX; block ship
- **Medium** — SHOULD FIX; include in next release
- **Low** — MAY fix; track as tech debt

## Tool availability check

Run the check appropriate to the project's ecosystem before auditing:

```bash
# Node.js / npm
command -v npm >/dev/null 2>&1 || echo "MISSING: npm"

# Node.js / yarn
command -v yarn >/dev/null 2>&1 || echo "MISSING: yarn"

# Node.js / pnpm
command -v pnpm >/dev/null 2>&1 || echo "MISSING: pnpm"

# Rust
command -v cargo-audit >/dev/null 2>&1 || echo "MISSING: cargo-audit — run: cargo install cargo-audit"

# Python
command -v pip-audit >/dev/null 2>&1 || echo "MISSING: pip-audit — run: pip install pip-audit"

# Go
command -v govulncheck >/dev/null 2>&1 || echo "MISSING: govulncheck — run: go install golang.org/x/vuln/cmd/govulncheck@latest"

# Ruby
command -v bundle-audit >/dev/null 2>&1 || echo "MISSING: bundle-audit — run: gem install bundler-audit"

# Java / Maven — requires internet access to fetch plugin
# No pre-check needed; mvn will fail with a clear error if misconfigured

# Java / Gradle — requires internet access
# No pre-check needed; gradlew will fail with a clear error if misconfigured

# .NET
command -v dotnet >/dev/null 2>&1 || echo "MISSING: dotnet SDK"
```

If the audit tool is unavailable, note the gap and advise installation — do not skip silently.
