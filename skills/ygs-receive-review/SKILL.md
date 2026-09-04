---
name: ygs-receive-review
description: Receive and evaluate code review feedback — before implementing suggestions, especially when feedback seems unclear, technically questionable, or conflicts with the design. Use after getting review comments from ygs-review-pr, ygs-code-review, or a human reviewer.
argument-hint: "[paste review findings or PR URL]"
---

# Receive Review

Read `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md` — you own the technical judgment, not just the implementation.

Code review is not a list of instructions to execute. It is a set of perspectives to evaluate. Your job is to implement what makes the code genuinely better, push back with reasoning on what doesn't, and ignore nothing.

## The Six-Step Pattern

Work through all feedback before acting on any of it.

**1. Read** — Read every comment before touching any code. The last comment may change how you interpret the first. Don't fix as you read.

**2. Understand** — For each finding: can you restate the concern in your own words without referencing the reviewer's exact phrasing? If not, ask for clarification before proceeding. Implementing something you don't understand produces code that doesn't fix what was intended.

**3. Verify** — Check the claim in the code. Is the reviewer correct? Is the code actually doing what they describe? Many review comments contain a factual claim about the code — confirm it before acting on it. Reviewers misread code.

**4. Evaluate** — Does implementing this suggestion make the code genuinely better? Apply YAGNI: reviewer suggestions for "future flexibility", "make it more extensible", or "professional standards" should be questioned unless there is evidence the flexibility will be needed. Complexity added for hypothetical future use is debt.

**5. Respond** — For findings you'll implement: say what you're doing and why briefly. For findings you're skipping or pushing back on: explain the technical reasoning explicitly. Never just say "you're right, I'll fix that" without stating what you're fixing and why the change improves the code.

**6. Implement** — Fix in priority order (see below). Commit after each logical group.

## Reviewer Trust Levels

**Human partner / team reviewer:** High trust on domain context and intent. Still verify technical claims in the code — even trusted reviewers misread.

**AI reviewer (ygs-review-pr, ygs-code-review, etc.):** High confidence on structural patterns; lower confidence on intent and business logic. Verify before implementing. AI reviewers are good at "this looks wrong" and weaker at "this is wrong given your requirements."

**External / unfamiliar reviewer:** Skeptical but thorough. Check every factual claim. Reasonable to push back with evidence. Quality varies widely.

## Forbidden Responses

Do not write these — they are social padding that demonstrate no understanding:

- "You're absolutely right!"
- "Great point!"
- "I hadn't thought of that!"
- "That's a really good suggestion"
- "Thanks for catching that"

State what you're fixing and why. That demonstrates understanding.

## When and How to Push Back

**When to push back:**
- The reviewer is factually wrong about what the code does
- The suggestion introduces YAGNI complexity (flexibility with no identified consumer)
- The suggestion conflicts with a documented design decision
- The suggestion would regress performance, correctness, or security
- The suggestion would violate the project's established patterns

**How to push back:**
> "I checked `[specific file:line]` and the code actually does X because Y. I'm keeping the current approach because Z. If I'm misunderstanding the concern, please tell me which scenario you're worried about."

Push back requires evidence and an alternative. "I don't think so" is not a push back.

## Implementation Priority Order

1. **Correctness / data loss / security** — fix immediately, these block everything
2. **API contracts and public interfaces** — affects other code, fix before moving on
3. **Test coverage gaps** — add missing tests
4. **Design and structural cleanups** — improve once correctness is established
5. **Style, naming, formatting** — batch into one commit at the end

## Completion

After all findings are triaged and implemented:

- Commit each logical group separately
- For findings you're deferring: create a task in `tasks/backlog/` rather than leaving the comment hanging
- Report **DONE** with: N findings implemented, M deferred (with reasons), P pushed back (with reasons)

---

References:
- `~/.claude/skills/you-got-skills/skills/shared/ownership-principles.md`
