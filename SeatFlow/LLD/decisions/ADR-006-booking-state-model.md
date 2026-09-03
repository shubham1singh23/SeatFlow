# ADR-006: Booking State Model — Table-Driven Guards, No State Pattern, No `CREATED` State

## Context
Booking needs a state machine; the "textbook" OOP answer is the State
design pattern.

## Problem
Does `BookingStatus` need per-state polymorphic behavior (which would
justify the State pattern), or just transition-legality rules?

## Options considered
- State Pattern: one class per status, each overriding transition methods.
- Table-driven guard function inside `BookingDomainService` validating
  (currentState, command) → allowed/new state, per
  `docs/lld/03-booking-lifecycle.md`.
- Include a persisted `CREATED` state prior to `HELD`.

## Decision
Table-driven guards, no State pattern; no persisted `CREATED` state
(creation and hold happen atomically in one transaction, so a
"created-not-held" state is never observable).

## Why we chose it
All 7 states share the same operations; they differ only in which
transitions are legal, not in how an operation behaves. A guard table is
more directly readable than 7 classes for that shape of problem. Removing
`CREATED` removes a state nothing could ever query, per
`docs/lld/03-booking-lifecycle.md`.

## Trade-offs
If a future requirement introduces genuinely different *behavior* per
state (not just legality), this decision should be revisited in favor of
State pattern.

## Consequences
State transitions are centralized in one place
(`BookingDomainService`), making the full legal-transition set auditable
in one file rather than scattered across state classes.

## Failure scenarios
Illegal transition attempts return `409 INVALID_STATE`, never silently
ignored or coerced.

## Future evolution
Revisit if per-state behavior (not just legality) is ever needed.
