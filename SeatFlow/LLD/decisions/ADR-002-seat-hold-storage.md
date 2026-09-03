# ADR-002: Seat Hold Storage — Merge into BookingItem, No Separate SeatHold Table

## Context
Common ticketing-system designs model a "seat hold" as its own entity/
table, separate from the eventual booking line item.

## Problem
Would a separate `seat_holds` table add anything for SeatFlow, given a
hold in this system is always created *as part of* a specific booking
attempt and never exists independently of one?

## Options considered
1. Separate `seat_holds` table, later "converted" into `booking_items` on
   confirmation.
2. Single `booking_items` table carrying hold fields (`holdToken`,
   `heldUntil`) alongside status, spanning the whole HELD→CONFIRMED/
   RELEASED lifecycle.

## Decision
Option 2. See `docs/lld/02-domain-model.md` for full reasoning.

## Why we chose it
No scenario in SeatFlow's current requirements needs a hold to outlive or
exist independently of its booking. A second table would duplicate state
and require a synchronization step ("convert" the hold row into a
booking-item row) that adds a second write, a second place for a bug to
live, and no additional guarantee — the unique index that protects the
core invariant works identically whether it's on one table or two.

## Trade-offs
If SeatFlow later needs holds that exist before a booking is chosen
(e.g. box-office holds), this decision must be revisited.

## Consequences
`BookingItem.status` carries more values (`HELD`, not just
`CONFIRMED`/`CANCELLED`) than a "pure" line-item table might otherwise
need — an explicit, accepted widening of one entity's responsibility in
exchange for removing a whole redundant table.

## Failure scenarios
N/A — this is a modeling decision, not a runtime-failure-prone mechanism.

## Future evolution
Promote hold state back into its own aggregate (`SeatHold`) if/when a
hold-without-a-booking feature is scoped.
