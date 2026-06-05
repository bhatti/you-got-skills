# Polyglot Test Runner

Auto-detect and run the project's test suite. Used by multiple skills that need to verify code health.

## Environment check

Before running, verify the tool is available:

```bash
command -v <tool> >/dev/null 2>&1 || echo "MISSING: <tool>"
```

If a required tool is missing: report **BLOCKED** with install instructions. If optional (linters, formatters): skip gracefully and note the gap.

## Detection and execution

```bash
[ -f Makefile ] && make test
[ -f package.json ] && npm test
[ -f Cargo.toml ] && cargo test
[ -f pytest.ini ] && pytest
[ -f go.mod ] && go test ./...
[ -f mix.exs ] && mix test
[ -f build.gradle ] && ./gradlew test
[ -f pom.xml ] && mvn test
```

## Build check (when needed)

```bash
[ -f Makefile ] && make build
[ -f package.json ] && npm run build
[ -f Cargo.toml ] && cargo build
[ -f go.mod ] && go build ./...
```

## Interpretation

- If tests **pass**: proceed with the workflow
- If tests **fail**: report **BLOCKED** — fix tests before continuing
- If no test runner detected: note the gap, proceed with manual verification
