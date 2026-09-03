# ADR-010: Explicit Strong-vs-Eventual Consistency Boundary

## Context
Not every part of SeatFlow needs the same consistency guarantee, and
treating everything as strongly consistent would unnecessarily couple
availability of low-stakes components (Notification) to the correctness-
critical path (Booking/Payment).

## Problem
Where exactly does the system require strong consistency, and where is
eventual consistency an explicit, deliberate choice rather than an
oversight?

## Options Considered

### Option 1: Strong consistency everywhere
Simplifies reasoning, but couples Notification/search-index availability
to Booking's write path unnecessarily, and makes every component as
"fragile" as the one that actually needs to be strict.

### Option 2: Eventual consistency everywhere
Would be unacceptable for seat allocation and payment state — the entire
premise of the project is that double-booking/double-charging must be
impossible, not "usually doesn't happen."

### Option 3: Explicit per-domain boundary
Strong consistency for seat state, bookings, and payments; eventual
consistency for notifications, catalog search/browse cache, and future
analytics.

## Decision
Option 3, stated explicitly here and referenced throughout the HLD
(originally declared in `01-requirements.md`).

## Why We Chose It
Every place strong consistency is claimed in this system (seat inventory,
booking confirmation, payment state) has a *specific, named* enforcement
mechanism (Postgres transactions + partial unique index, ADR-002/004).
Every place eventual consistency is accepted (notifications, catalog
cache) has a *specific, named* reason it's safe to be stale/delayed
(re-checked at the point that actually matters, or simply non-critical).
Nothing in the system is "consistent by convention" without a named
mechanism behind it.

## Trade-offs
- Requires discipline to keep the boundary honest — every new feature
  must be explicitly classified rather than defaulting to "just make it
  eventually consistent, it's simpler."

## Consequences
- This ADR is the reference point for every other document's consistency
  claims — `07`, `09`, `10`, `11`, and `15` all trace back to this
  boundary rather than re-deriving it independently.

## Failure Scenarios
N/A — this is a policy decision; concrete failure behavior per domain is
covered in `11-reliability-and-failure-handling.md`.

## Future Evolution
If a future feature (e.g. loyalty points, waitlists) is added, it must be
explicitly classified against this same boundary before implementation,
not left ambiguous.
