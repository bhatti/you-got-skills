# Design Principles — Canonical Reference

Single source of truth for SOLID, package cohesion, stability metrics, hotspot coupling, and readability.
**Do not duplicate these definitions** in individual skills — reference this file instead.

Referenced by: `ygs-pr-risk`, `ygs-code-review`, `ygs-review-deep/specialists/architecture.md`,
`ygs-pr-audit`, `ygs-codebase-audit`, `shared/quality-checklist.md`.

---

## 1. SOLID Checklist

Apply per class/module touched in the diff. Each violation is a candidate finding.

| Principle | What to check in the diff | Violation signals | Severity |
|-----------|--------------------------|-------------------|----------|
| **SRP** Single Responsibility | Does this class/module now have more than one distinct reason to change? | "and" in class name; unrelated public methods; constructor takes 5+ unrelated deps; test setup requires unrelated fixtures | SHOULD (2 concerns), MUST (3+) |
| **OCP** Open/Closed | Does new behaviour require modifying existing classes rather than extending? | `if type == X / elif type == Y` chains that will grow; switch on type enum in core logic | SHOULD |
| **LSP** Liskov Substitution | Does a subclass weaken a precondition or strengthen a postcondition of the parent? | Overridden method throws `NotImplementedError`; subclass silently ignores a parameter the parent requires; narrower return type rejected by parent callers | MUST |
| **DIP** Dependency Inversion | Does a high-level module import a concrete low-level type directly? | Domain/service layer imports `from infra.db import …` or `from storage import …`; constructor accepts a concrete class where an interface/protocol exists | MUST |
| **ISP** Interface Segregation | Does an interface force implementors to stub unrelated methods? | Interface with 8+ methods where callers use 2–3; implementors that raise `NotImplementedError` on half the interface | SHOULD |

**LSP, OCP detection note**: these require reading the type hierarchy or surrounding callers — confirm
by checking 2–3 call sites before flagging. Do not flag from the diff alone without evidence.

---

## 2. Package Cohesion (CCP, CRP, REP)

These apply at **module/package** level, not individual class level. Check when diff touches module
structure, new packages, or new public exports.

### CCP — Common Closure Principle
Classes that change together should live together. A violation = files in different modules that always
appear in the same commit.

**Detection** (use git log for temporal coupling):
```bash
# Files that co-change with the file you're reviewing (last 50 commits)
git log --oneline -50 --name-only -- <path/to/changed_file> \
  | grep "^[a-z]" | sort | uniq -c | sort -rn | head -20
# Files appearing 5+ times alongside your file are temporally coupled
```

Flag: if two files co-change in >30% of commits but live in different modules → CCP violation (SHOULD).

### CRP — Common Reuse Principle
Classes not reused together should not be grouped. A violation = utility module that mixes unrelated concerns.

**Signal**: a module with imports from 4+ distinct business domains, or a `utils/` or `helpers/` file that
serves callers from completely different layers.

### REP — Reuse/Release Equivalency
A module should only contain code that is released together. Flag: production code mixed with internal
tooling scripts in the same package/namespace.

---

## 3. Stability Metrics (ADP, SDP, SAP)

These formulas let you assign an objective score to module stability. Apply to **changed modules** in
per-PR reviews; apply to **all modules** in codebase audits.

### Formulas

| Metric | Formula | Range | Meaning |
|--------|---------|-------|---------|
| **Ce** (efferent coupling) | Count distinct external modules this module imports | — | "Depends on how many" |
| **Ca** (afferent coupling) | Count distinct modules that import this module | — | "How many depend on it" |
| **I** (Instability) | Ce / (Ca + Ce) | 0–1 | 0 = very stable; 1 = very unstable |
| **A** (Abstractness) | abstract_classes / total_classes | 0–1 | 0 = fully concrete; 1 = fully abstract |
| **D** (Distance from main sequence) | \|A + I − 1\| | 0–1 | 0 = ideal; 1 = zone of pain or uselessness |

### Thresholds

| Metric | Healthy 🟢 | Warning 🟡 | Flag 🔴 | Action |
|--------|-----------|-----------|--------|--------|
| I (shared/core modules) | < 0.4 | 0.4–0.6 | > 0.6 | Reduce outbound deps or add abstraction layer |
| I (leaf/adapter modules) | 0.7–1.0 expected | — | < 0.3 (over-stabilised leaf) | Correct — no action |
| D (distance) | < 0.3 | 0.3–0.5 | > 0.5 | Zone of pain (stable+concrete) or uselessness (unstable+abstract) |
| ADP cycles | 0 | — | Any cycle | MUST: resolve with DIP or extract shared abstraction |

### Detection heuristics (no tooling required)

```bash
# Ce: count distinct modules this module imports from
grep -rn "^from\|^import" <module_dir>/ --include="*.py" 2>/dev/null \
  | awk '{print $2}' | cut -d'.' -f1 | sort -u \
  | grep -v "__\|test\|$(basename <module_dir>)" | wc -l

# Ca: count distinct other modules that import this module
MODULE=$(basename <module_dir>)
grep -rn "from ${MODULE}\b\|import ${MODULE}\b" \
  --include="*.py" --include="*.go" --include="*.ts" . 2>/dev/null \
  | grep -v "<module_dir>/" | awk -F: '{print $1}' \
  | xargs -I{} dirname {} | sort -u | wc -l

# Go: same via go list
go list -f '{{.Imports}}' ./... 2>/dev/null | tr ' ' '\n' | sort | uniq -c | sort -rn | head -20

# ADP: cycle detection (Python)
python3 -c "
import ast, os
deps = {}
for root, _, files in os.walk('.'):
    for f in files:
        if f.endswith('.py') and '__pycache__' not in root:
            path = os.path.join(root, f)
            mod = path.replace('/', '.').replace('.py', '').lstrip('.')
            try:
                tree = ast.parse(open(path).read())
                deps[mod] = [n.module or '' for n in ast.walk(tree)
                             if isinstance(n, (ast.Import, ast.ImportFrom)) and hasattr(n, 'module')]
            except: pass
for a, bs in deps.items():
    for b in bs:
        if b in deps and any(a.startswith(x or 'NOMATCH') for x in deps.get(b, [])):
            print(f'CYCLE: {a} <-> {b}')
" 2>/dev/null | head -10
```

### Stable Dependency Principle (SDP)
No module should depend on a module that is less stable than itself.
**Flag**: module with I=0.2 importing from module with I=0.8 → dependency flows wrong direction.

### Stable Abstractions Principle (SAP)
Stable modules (low I) should be abstract (high A).
**Flag**: I=0.1 (very stable) + A=0.0 (fully concrete) → zone of pain; hard to extend without modification.

---

## 4. Hotspot Coupling Analysis

A file is a **design hotspot** when it is simultaneously:
1. High churn (top decile by commit count) — see `shared/sloppiness-metrics.md` for the pipeline
2. High complexity (CC > 10 in at least one function)
3. High instability (I > 0.5 — many outbound deps, few inbound)

Such files combine maximum defect risk (churn×CC) with maximum change propagation risk (high I). They
are the most urgent refactoring targets.

**Cross-reference in per-PR reviews:**
```bash
# Quick check: is the changed file a top-30 churn file?
git log --oneline --name-only -- <changed_file> 2>/dev/null | grep -v "^[0-9a-f]" | wc -l
# > 20 commits touching this file in recent history → hotspot candidate
```

Flag as `[DESIGN] Hotspot coupling` when: the file being changed is both a churn hotspot AND has
high instability — further changes here propagate widely and are high-risk.

---

## 5. Readability Dimension

Readability = how easily a new reader can understand and correctly predict the code's behaviour,
distinct from naming correctness (covered in `maintainability.md`) and cognitive load (CC thresholds).

### Five sub-dimensions (score each 0–2)

| Sub-dimension | 2 — Good | 1 — Ok | 0 — Poor |
|---------------|----------|--------|----------|
| **Scannability** | Function signature + first line tells you the full story; no surprise in the body | Body mostly follows the signature's promise | Must read entire body to understand what it does |
| **Locality of information** | All context needed is in the same file or 1 call away | Need to trace 2–3 hops through the call graph | Understanding requires reading 4+ distant files |
| **Abstraction consistency** | Single function operates at one abstraction level throughout | Minor mixing of high/low level | High-level intent mixed with low-level mechanics in same function |
| **Progressive disclosure** | Happy path is immediately obvious; edge cases are handled but not foregrounded | Happy path requires light scanning | Must parse complex branching to find the normal flow |
| **Predictability** | Side effects visible at call site (naming, type signature, or doc); no hidden mutations | Side effects documented but not visible from caller | Innocent-looking call has surprising mutations or I/O side effects |

**Readability score** = Σ(sub-dimension scores) / 10 → range [0.0, 1.0]; multiply by 10 for display (0–10).
Benchmark: > 7 healthy; 5–7 warning; < 5 flag SHOULD.

### Per-PR readability assessment
Score only the **changed functions** (not unchanged context). Average the scores.
If all changed functions score ≥ 7: report readability as 🟢.
If any changed function scores ≤ 4: report a readability finding with the specific sub-dimension that failed.

---

## Anti-patterns Quick Reference

| Anti-pattern | Principle violated | Severity |
|---|---|---|
| God class / God module | SRP | MUST (CC > 15 or > 5 unrelated concerns) |
| Concrete dependency in domain layer | DIP | MUST |
| Fat interface (8+ methods, callers use 2) | ISP | SHOULD |
| `if type == X` dispatch chain | OCP | SHOULD |
| Subclass raises NotImplementedError | LSP | MUST |
| Highly stable+fully concrete module | SAP | SHOULD |
| High-churn + high-instability file | Hotspot coupling | SHOULD |
| Temporal coupling across modules | CCP | SHOULD |
| Low I stable module depending on high I unstable module | SDP | SHOULD |
