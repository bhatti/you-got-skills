---
name: ygs-uat
description: User Acceptance Testing — validates user stories against PRD acceptance criteria, exercises end-to-end flows, and produces a pass/fail report with a clear failure path.
argument-hint: "[--prd <path>] [--story <id>]"
---

# User Acceptance Testing

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — you own the UAT quality.

**Iron law: UAT validates behavior, not code. Exercise the system as a user would.**

## Step 1: Locate PRD and extract acceptance criteria

Follow `shared/docs-discovery.md` to find the PRD.

For each user story in the PRD, extract the Given/When/Then scenarios (EARS patterns). If the PRD uses informal language, convert to Given/When/Then before proceeding.

Reference: `~/.claude/skills/you-got-skills/skills/shared/ears-patterns.md`

If no PRD exists: ask the user to describe the user stories explicitly. Do not invent acceptance criteria.

## Step 2: Verify the system is testable

Run the test suite to confirm baseline health:

Follow `~/.claude/skills/you-got-skills/skills/shared/test-runner.md`.

If tests are failing before UAT begins, report **BLOCKED** — UAT on a broken build produces meaningless results.

## Step 3: Trace and exercise each acceptance scenario

For each Given/When/Then scenario:

**Trace the code path:**
- Identify the entry point (API endpoint, UI action, CLI command, event handler)
- Follow the path through to the expected output
- Identify any branches not covered by the scenario

**Exercise the flow:**
- Use the actual interface (not internal functions) — same path a user would take
- Use realistic data: long names, special characters, edge-case values
- Deliberately try to break each step: missing fields, wrong types, concurrent actions

**Check user-facing quality:**
- Does the user get appropriate feedback at each step?
- Are error messages user-friendly (no stack traces, no internal identifiers)?
- Can the user recover from mistakes without losing their work?
- Is performance acceptable? (Page/response should not feel slow)

## Step 4: Test failure paths (not just happy path)

For each P0 EARS `If/Then` requirement from the PRD:
- Trigger the failure condition deliberately
- Confirm the system responds as specified (error message, fallback, graceful degradation)
- Confirm the user is not left in an inconsistent state

A happy-path-only UAT is incomplete. The user will encounter failure paths in production.

## Step 5: Report

For each user story:

| Story | Scenario | Result | Evidence | Notes |
|-------|----------|--------|----------|-------|
| | | PASS / FAIL / PARTIAL | Code path / output observed | |

**PASS** — Behavior matches acceptance criteria exactly.
**FAIL** — Behavior diverges from acceptance criteria. Name the gap precisely.
**PARTIAL** — Works but with friction, missing edge-case coverage, or degraded UX.

**On FAIL or PARTIAL:** create a task immediately:
- File: `tasks/backlog/uat-<story-id>-<date>.md`
- Include: failing scenario, expected vs actual, steps to reproduce

**Overall verdict:**
- Is the feature ready for release? (All P0 stories PASS)
- What are the blockers for user satisfaction?
- What is deferred (PARTIAL or low-priority FAIL)?

Report **DONE** if all P0 stories PASS, **DONE_WITH_CONCERNS** if any P1/P2 fail or PARTIAL results exist, **BLOCKED** if the system cannot be exercised.

Suggest: `/ygs-investigate` for any FAIL with unclear root cause.
