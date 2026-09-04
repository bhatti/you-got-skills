# Verification Gate

Iron law for all skills: no completion claims without fresh verification evidence.

## The Rule

**Identify the verification command → run it → read full output → confirm the claim matches output → only then assert success.**

A claim made without running a command in the same response is a guess. A guess is not a completion.

## What Each Claim Requires

| Claim | What is required | What is insufficient |
|-------|-----------------|---------------------|
| "Tests pass" | Run test command, show output with pass count and zero failures | "Tests should pass", previous run, CI status from earlier |
| "Feature works" | Exercise the actual behavior (CLI invocation, HTTP request, dev server run) and show the output | "The logic looks right", "tests cover it" |
| "No errors" | Show full log output with no ERROR/FATAL lines | "I didn't see any errors", skipping the run |
| "Bug is fixed" | Reproduce the original failure first, then show it no longer occurs | "I fixed the code path", "the logic is correct now" |
| "No regressions" | Full test suite run, output shown | "I only changed X so Y shouldn't break" |
| "Build succeeds" | Show build command output with success exit | "Should compile fine", "no syntax errors visible" |

## Agent Delegation Rule

When another agent (subagent, ygs-review-deep specialist, etc.) reports success: **trust the VCS diff, not the agent's success report.** An agent can claim success while having produced no actual changes. Review the actual diff before accepting a completion.

```bash
git diff HEAD~1..HEAD --stat   # confirm files actually changed
git diff HEAD~1..HEAD          # confirm changes match the claim
```

## Red Flags

- "This should work now" — should is not does
- "I believe the tests pass" — belief is not evidence
- "The logic looks correct" — logic review is not test execution
- Reporting DONE before running the verification command
- Running a command but not reading the output before claiming success

## Enforcement

Skills that load this file must include a verification step that produces command output before any DONE/DONE_WITH_CONCERNS signal. The output must be in the same response as the claim.
