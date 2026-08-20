---
name: ygs-review-deep
description: Deep multi-specialist PR review — runs the standard 4-domain review (correctness, security, API, SRE) then adds 3 specialist passes (performance, testing quality, architecture/maintainability). Use for high-risk PRs or large diffs.
argument-hint: "<pr-url-or-number>"
---

# Deep PR Review

For shared review protocol (severity classification, finding format, verdict mapping), read:
`~/.claude/skills/you-got-skills/skills/shared/review-scaffold.md`

This skill extends `ygs-review-pr` with 3 additional specialist passes. Run the standard review first, then the specialist passes, then synthesize.

---

## Phase 1: Standard Review (4 domains)

Follow all steps in `~/.claude/skills/you-got-skills/skills/ygs-review-pr/SKILL.md` to complete the standard review. Collect findings into a working list tagged by domain.

---

## Phase 2: Specialist Passes (3 additional domains)

After the standard review, run these 3 passes on the same diff. Each pass is focused — skip categories that clearly don't apply to the diff.

### Pass 5 — Performance

Focus on:
- **Hot paths**: are expensive operations (DB queries, network calls, serialization) in loops or called on every request?
- **N+1 queries**: does any new code fetch in a loop what could be batched in one query?
- **Memory allocation**: unnecessary object creation in hot paths, large intermediate collections, missing streaming for large datasets
- **Missing caching**: repeated computation of the same value, DB lookups that could be cached with a short TTL
- **Connection pool exhaustion**: creating new connections per request instead of reusing a pool
- **Benchmark regressions**: if benchmarks exist, does the diff touch their critical paths?

Severity guidance:
- CRITICAL: DB query or network call inside a loop on unbounded data
- HIGH: O(n²) algorithm on inputs that can be large in production
- MEDIUM: repeated computation that could be cached, unnecessary allocations

### Pass 6 — Testing Quality

Focus on:
- **Coverage of new code**: does every new code path have at least one test? Are error paths tested?
- **Test design**: do tests verify behavior (what the code does) or implementation (how it does it)? Tests tightly coupled to implementation details break on safe refactors.
- **Empty or trivial tests**: tests that always pass regardless of the code (e.g., `assert True`, testing a mock's own behavior)
- **Flaky test patterns**: `time.Sleep`, fixed ports, shared mutable state between tests, non-deterministic ordering
- **Missing edge case tests**: boundary conditions (empty input, nil/null, maximum size), concurrent access, idempotency
- **Test-to-implementation coupling**: is the test calling private methods, or checking internal state that shouldn't be observable?

Severity guidance:
- CRITICAL: new security or data-integrity code path with no test
- HIGH: entire new feature with no integration test
- MEDIUM: error path untested, flaky pattern introduced

### Pass 7 — Architecture & Maintainability

Focus on:
- **Module coupling**: does this change introduce a new import cycle or create a dependency from a lower-level module to a higher-level one?
- **Abstraction level**: is the new code at the right level of abstraction? Too deep (everything is an interface) or too shallow (duplicated logic that should be extracted)?
- **Naming clarity**: do names reveal intent? Would a new team member understand what a function/variable does from its name alone, without reading the body?
- **Cognitive load**: can a reader follow the logic without holding more than 5-7 concepts in mind? Flag deeply nested conditions, long functions (>50 lines), or functions doing multiple unrelated things.
- **Duplication**: does this introduce a 3rd+ copy of logic that should be extracted? (Two copies is sometimes fine; three is a pattern.)
- **Fit with existing patterns**: does the new code introduce a different style, framework, or pattern without clear justification? New paradigms require context.
- **Non-obvious logic**: is there a comment explaining WHY (not what) for any tricky invariant, workaround, or subtle behavior?

Severity guidance:
- HIGH: new circular dependency, function >100 lines with multiple concerns
- MEDIUM: naming that obscures intent, missing explanation for a non-obvious invariant
- LOW: minor naming suggestions, style preferences

---

## Phase 3: Synthesize

1. Merge all findings from Phase 1 (4 domains) and Phase 2 (3 specialist passes)
2. Deduplicate: if two passes flag the same file+line for the same reason, keep the higher-severity finding
3. Rank: CRITICAL → HIGH → MEDIUM → LOW, then by file path within each tier
4. Write the merged `findings.json` (same schema as `ygs-review-pr`)
5. Write `reports/review.md` with a structured summary:
   - One-paragraph overall assessment
   - Table: domain → finding count by severity
   - Full findings list, ranked

Output ONLY this JSON on the last line:
`{"status":"DONE","findings_count":<N>,"verdict":"<APPROVE|REQUEST_CHANGES|COMMENT>","summary":"<one sentence>"}`
