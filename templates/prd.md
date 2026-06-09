# PRD: [Feature Name]

**Date:** YYYY-MM-DD
**Author:** [Name]
**Status:** Draft | Review | Approved

## Problem Statement
<!-- What problem exists today? One paragraph. Why now? -->

## Target User
<!-- Who benefits? Be specific about persona and context. -->

## Success Criteria
<!-- 3-5 measurable outcomes that define "done" -->

## Capabilities

### New Capabilities
<!-- List new behaviors/features being introduced -->
- 

### Modified Capabilities
<!-- List existing behaviors whose requirements are changing -->
- 

### Removed Capabilities (if any)
<!-- Behaviors being deprecated. Mark BREAKING if applicable. -->
- 

## Requirements

Use RFC 2119 keywords: MUST (P0), SHOULD (P1), MAY (P2)

### Requirement: [Name]
The system MUST/SHOULD/MAY [observable behavior]

<!-- EARS notation for behavioral precision (see skills/shared/ears-patterns.md):
  THE SYSTEM SHALL [behavior]                                    (ubiquitous — always active)
  WHEN [trigger], THE SYSTEM SHALL [response]                    (event-driven)
  WHILE [state], THE SYSTEM SHALL [behavior]                     (state-driven)
  WHERE [feature is included], THE SYSTEM SHALL [behavior]       (optional feature)
  IF [fault/unwanted trigger], THEN THE SYSTEM SHALL [response]  (unwanted behaviour)
  WHILE [state], WHEN [trigger], THE SYSTEM SHALL [response]     (complex — state + event)

  Rule: every MUST requirement needs at least one IF/THEN covering its failure mode.
-->

#### Scenario: [Happy path]
- **GIVEN** [precondition]
- **WHEN** [action]
- **THEN** [expected outcome]

#### Scenario: [Edge case]
- **GIVEN** [precondition]
- **WHEN** [action]
- **THEN** [expected outcome]

## Non-Goals
<!-- Explicitly out of scope for this effort -->

## Impact
<!-- Affected systems, APIs, teams, dependencies -->

## Constraints
<!-- Performance, compliance, compatibility, timeline requirements -->

## Open Questions
<!-- Unresolved decisions that need input -->
