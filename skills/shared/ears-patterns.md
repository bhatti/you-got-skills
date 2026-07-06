# EARS Patterns

Canonical reference for EARS (Easy Approach to Requirements Syntax). All skills reference this file — do not duplicate pattern definitions elsewhere.

Source: https://alistairmavin.com/ears/

## What EARS is

A lightweight constraint on natural-language requirements that makes them unambiguous, testable, and traceable. Each requirement must match exactly one of the six patterns below. The pattern keyword signals the temporal relationship between condition and system response.

## The Six Patterns

### 1. Ubiquitous
Always active — no trigger or precondition.

**Template:** `The <system> shall <system response>`

**When to use:** Invariants, always-on behaviors, baseline capabilities.

**Example:** `The API gateway shall enforce TLS 1.2 or higher on all connections.`

---

### 2. Event-Driven (keyword: `When`)
Triggered by a discrete event or stimulus.

**Template:** `When <trigger>, the <system> shall <system response>`

**When to use:** Response to user action, external event, message arrival, or state change notification.

**Example:** `When the user submits a payment form, the system shall validate the card number within 200ms.`

---

### 3. State-Driven (keyword: `While`)
Active only while the system is in a particular state.

**Template:** `While <precondition(s)>, the <system> shall <system response>`

**When to use:** Ongoing behavior that is conditional on system or context state persisting over time.

**Example:** `While the session is authenticated, the system shall refresh the access token every 15 minutes.`

---

### 4. Optional Feature (keyword: `Where`)
Applies only when an optional capability is present or configured.

**Template:** `Where <feature is included>, the <system> shall <system response>`

**When to use:** Behavior conditional on deployment configuration, licensed feature, or hardware capability. Not a runtime state — a build-time or configuration-time condition.

**Example:** `Where the analytics module is enabled, the system shall emit a page-view event on each route change.`

---

### 5. Unwanted Behaviour (keywords: `If` / `Then`)
Defines the system's response to an undesired or error condition.

**Template:** `If <trigger>, then the <system> shall <system response>`

**When to use:** Error handling, fault tolerance, invalid input, constraint violation, timeout. Every non-happy-path outcome needs one of these.

**Example:** `If the upstream service does not respond within 5 seconds, then the system shall return a 503 with a Retry-After header.`

---

### 6. Complex (combined keywords)
Combines a state precondition with an event trigger, or multiple patterns.

**Template:** `While <precondition(s)>, when <trigger>, the <system> shall <system response>`

**When to use:** Only when both a persisting state AND a discrete event are necessary to trigger the response. Avoid overuse — if you can express it with a single pattern, do so.

**Example:** `While the order is in PENDING state, when the payment confirmation event is received, the system shall transition the order to CONFIRMED and emit an order-confirmed event.`

---

## Pattern Selection Guide

```
Is it always active (no condition, no trigger)?
  → Ubiquitous

Is it triggered by a discrete event?
  Does it also require the system to be in a specific persisting state?
    Yes → Complex (While + When)
    No  → Event-Driven (When)

Is it an ongoing behavior while a state persists (no discrete trigger)?
  → State-Driven (While)

Is it an error, fault, or unwanted condition?
  → Unwanted Behaviour (If/Then)

Is it conditional on an optional feature or deployment config?
  → Optional Feature (Where)
```

## Validation Checklist

Apply these checks to every behavioral requirement in a PRD, TRD, or acceptance criterion:

- [ ] Does it match exactly one EARS pattern? (If it matches two, split it.)
- [ ] Is the system name explicit and consistent?
- [ ] Is the trigger/precondition specific and observable (not vague)?
- [ ] Is the system response a single, testable action? (Multiple responses → multiple requirements)
- [ ] Are unwanted behaviours covered for each happy-path requirement? (If no `If/Then` exists for a `When`, ask why.)
- [ ] Are optional features marked `Where`, not buried in conditionals?

## Integration with RFC 2119

EARS defines temporal structure; RFC 2119 defines obligation strength. Combine them:

| RFC 2119 | EARS pattern | Meaning |
|----------|-------------|---------|
| `MUST` (P0) | `When X, the system shall Y` | Mandatory event-response |
| `SHOULD` (P1) | `While X, the system shall Y` | Strong state-driven behavior |
| `MAY` (P2) | `Where X is included, the system shall Y` | Optional feature behavior |

**Rule:** `MUST` requirements need at least one `If/Then` covering failure modes. If a P0 requirement has no unwanted-behaviour counterpart, it is incomplete.
