# Completion Signals

Canonical definitions for DONE, DONE_WITH_CONCERNS, and BLOCKED.
All skills reference this — do not redefine these terms inline.

## DONE

The work is complete and verified. All of the following are true:
- Acceptance criteria are met (tested, not assumed)
- Tests pass (run them, don't just read the code)
- No known regressions introduced
- Output is in the expected format/location

**Do not report DONE if any criterion is unverified.** "It should work" is not DONE.

## DONE_WITH_CONCERNS

The primary objective is met, but there are known issues worth flagging:
- The feature works but with a performance or UX tradeoff
- Tests pass but coverage is thin for a specific edge case
- The fix is correct but reveals a structural problem that should be addressed separately
- A recommendation was made but not acted on due to scope

Always name the concern explicitly. "DONE_WITH_CONCERNS" with no description is not useful.

## BLOCKED

Progress cannot continue without external input or a decision. Use when:
- A required tool, credential, or environment is unavailable
- A question must be answered before the correct path is knowable
- A dependency (other task, person, team) must complete first
- The root cause cannot be determined with available information

Always state:
1. What specifically is blocking (not "something is wrong")
2. What was tried
3. What would unblock progress

**BLOCKED is not a failure state** — it is an honest signal that stops wasted effort. Report it early, not after exhausting all options in silence.
