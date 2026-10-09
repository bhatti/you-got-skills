# Specialist: Architecture & Design

Review scope: does the structure of this change belong in the existing design, or does it introduce technical debt?

For functional design principles, read `~/.claude/skills/you-got-skills/skills/shared/functional-design.md`.
For quality rules (cyclic deps, modular boundaries, CC thresholds), read `~/.claude/skills/you-got-skills/skills/shared/quality-checklist.md` sections 3 and 4.

## Module boundaries

- Does this change introduce a new import cycle?
- Does a dependency flow in the wrong direction (e.g., domain importing from infrastructure)?
- Is a new module necessary, or is this an existing module that should be extended?
- Deletion test: if this new abstraction were removed, would its complexity disperse to N callers? If yes, it is earning its keep. If not, it is a shallow wrapper.

## Abstraction level

- Is the new code at the right level of abstraction?
- Too deep: everything is an interface with a single implementation — over-engineered
- Too shallow: duplicated logic across callers that should be extracted — under-engineered
- Does this introduce a 3rd+ copy of logic that should be unified?

## Cohesion with existing design

- Does the new code follow the naming conventions, module structure, and patterns of the surrounding code?
- Is a new paradigm or framework introduced without justification? New patterns require context.
- Does the SDK/wrapper layer contain business logic that belongs in the underlying framework?

## Circular dependencies

- New circular module reference introduced — resolve with proper abstraction, not a trait workaround
- Dependency inversion missing where it would break the cycle cleanly

**Detection (run for any diff touching import lists):**
```bash
# Go — build-time cycle detection
go build ./... 2>&1 | grep "import cycle" | head -10

# Python — grep for mutual imports (A imports B and B imports A)
python3 -c "
import ast, os, sys
deps = {}
for root, _, files in os.walk('.'):
    for f in files:
        if f.endswith('.py') and '__pycache__' not in root:
            path = os.path.join(root, f)
            mod = path.replace('/', '.').replace('.py', '').lstrip('.')
            try:
                tree = ast.parse(open(path).read())
                deps[mod] = [n.module or '' for n in ast.walk(tree) if isinstance(n, (ast.Import, ast.ImportFrom)) and hasattr(n, 'module')]
            except: pass
for a, bs in deps.items():
    for b in bs:
        if b in deps and any(a in (x or '') for x in deps[b]):
            print(f'CYCLE: {a} <-> {b}')
" 2>/dev/null | head -10

# JS/TS — madge
npx madge --circular --extensions ts,js src/ 2>/dev/null | head -20
```

## Interface design (deep modules)

- Small, stable interface with a rich implementation = high leverage
- Large interface with thin implementation = shallow pass-through (anti-pattern)
- Public API exposes only what callers need; internal details stay hidden

## Data contracts

- New cross-boundary data types defined at the appropriate contract layer
- Existing entity and ID types used consistently — no ad-hoc construction or parsing where a canonical form already exists in the codebase
- No stringly-typed parameters where a typed enum or newtype would serve

## SOLID scan

Read `~/.claude/skills/you-got-skills/skills/shared/design-principles.md` (Section 1) for the full SOLID
checklist, violation signals, and severity table. Apply to each class/module touched in the diff:

- **SRP**: does this change give a class/module more than one distinct reason to change?
  Look for: unrelated public methods grouped together, constructor taking 5+ unrelated dependencies,
  class name containing "and" or "manager" or "handler" that does everything.
- **OCP**: does the new behaviour require editing an existing class instead of extending it?
  Look for: new `if type == X` branches added to core logic, switch statements on type enums growing.
- **LSP**: does a subclass weaken a precondition or strengthen a postcondition?
  Look for: overridden method that raises `NotImplementedError` or ignores a required parameter.
- **DIP**: does a high-level module import a concrete low-level type directly?
  Look for: domain/service layer importing from `infra.*`, `db.*`, `storage.*`, `adapter.*` directly.
- **ISP**: does an interface force implementors to stub unrelated methods?
  Look for: interface with 8+ methods where diff's caller only calls 2–3 of them.

## Package stability

Read `~/.claude/skills/you-got-skills/skills/shared/design-principles.md` (Sections 2–3) for package
cohesion rules (CCP, CRP, REP) and stability metric formulas (I, A, D).

For each module changed in the diff, estimate:
1. **Ce** — count distinct external modules it imports (grep `^from|^import` for Python; `import "` for Go)
2. **Ca** — count distinct modules that import it (grep for the module name in sibling modules)
3. **I** = Ce / (Ca + Ce)

Flag:
- I > 0.7 for a shared/core module → SHOULD (too many outbound deps, too fragile)
- A stable module (I < 0.3) that is fully concrete (no interfaces) → SHOULD (SAP violation — zone of pain)
- Any confirmed ADP cycle → MUST

For **hotspot coupling**: cross-reference with churn data per `shared/design-principles.md` Section 4.
If a changed file is both a top-churn file AND has I > 0.5, flag `[ARCH] Hotspot coupling` as SHOULD.

```bash
# Quick Ce estimate for a changed Python module
MODULE_DIR="<path/to/changed/module>"
grep -rn "^from\|^import" "${MODULE_DIR}/" --include="*.py" 2>/dev/null \
  | awk '{print $2}' | cut -d'.' -f1 | sort -u \
  | grep -v "__\|test" | wc -l

# Quick Ca estimate
MODULE=$(basename "${MODULE_DIR}")
grep -rn "from ${MODULE}\b\|import ${MODULE}\b" \
  --include="*.py" --include="*.go" --include="*.ts" . 2>/dev/null \
  | grep -v "${MODULE_DIR}/" | awk -F: '{print $1}' | xargs -I{} dirname {} | sort -u | wc -l
```

## Design Quality Metrics table

At end of this specialist pass, emit the **Design Quality Metrics table** from
`~/.claude/skills/you-got-skills/skills/shared/design-metrics-table.md`.
Fill in values for: SRP violations, DIP violations, cyclic deps (ADP), layer violations,
avg instability (I), hotspot coupling, readability score (if scored in maintainability pass), ISP fat interfaces.

## Severity guidance

| Severity | Example |
|----------|---------|
| MUST | New circular dependency (ADP violation), dependency direction inversion |
| MUST | Business logic in SDK/wrapper layer instead of framework |
| MUST | DIP violation: domain layer importing concrete infrastructure type |
| MUST | LSP violation: subclass raises NotImplementedError on inherited method |
| SHOULD | SRP violation: class/module has 2+ distinct change reasons |
| SHOULD | Abstraction with a single consumer, shallow pass-through module |
| SHOULD | New pattern introduced without precedent or explanation |
| SHOULD | I > 0.7 for a core/shared module (too many outbound deps) |
| SHOULD | SAP violation: I < 0.3 (stable) + A ≈ 0 (fully concrete) — zone of pain |
| SHOULD | ISP violation: fat interface forcing stubbed unrelated methods |
| SHOULD | Hotspot coupling: high-churn file with I > 0.5 |
| MAY | Minor naming inconsistency with surrounding code |
