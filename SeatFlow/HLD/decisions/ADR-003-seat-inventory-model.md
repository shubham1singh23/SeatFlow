# ADR-003: Separate Physical Seats (Catalog) From Show-Specific Inventory (Booking)

## Context
A physical seat (e.g. Screen 3, Row F, Seat 12) exists independent of any
particular show, but its *booking state* (available/held/booked) only
makes sense in the context of one specific show.

## Problem
Should "a seat" be a single entity/table shared across services, or two
distinct concepts owned by different services?

## Options Considered

### Option 1: Single shared `seats` table with a status column
One row per physical seat; status mutated per active show. Fails
immediately: a seat is AVAILABLE for the 3pm show and BOOKED for the 7pm
show *simultaneously* — a single status column per physical seat cannot
represent that.

### Option 2: One row per (seat, show) in a single service
Correct data model, but which service owns it? If Catalog owns it, the
write-heavy, lock-sensitive booking table lives inside the
availability-optimized, cache-heavy service — wrong operational profile.

### Option 3: Physical seat (Catalog) + ShowSeatInventory (Booking), separate services
Physical layout is static reference data owned by Catalog; per-show
booking state is owned by Booking, keyed by `(show_id, seat_id)`.

## Decision
Option 3 — physical seats in Catalog, `show_seat_inventory` in Booking.

## Why We Chose It
This aligns data ownership with **consistency needs and write patterns**,
not just entity modeling: physical seats change rarely and are read
constantly (Catalog's profile); show-seat state changes constantly under
contention and must be transactionally guarded (Booking's profile).
Keeping them in the same service/table would force Booking's
concurrency-sensitive workload to share infrastructure with Catalog's
high-read-volume, cache-friendly workload.

## Trade-offs
- Booking Service needs to know about seats that "exist" in Catalog —
  handled by snapshotting seat identity/category/price into
  `show_seat_inventory` when a show is published, so Booking's hot path
  never needs a live cross-service call to Catalog.
- Slight duplication of seat identifiers across two services — accepted,
  because it decouples their failure/scaling profiles completely.

## Consequences
- `show_seat_inventory` rows are created in bulk when a show is
  published (one row per physical seat on that screen).
- Catalog going down does not block existing shows from being booked.

## Failure Scenarios
- Catalog outage during show creation: new shows can't be published, but
  already-published shows' booking flow is entirely unaffected (Booking
  already has its own copy of what it needs).

## Future Evolution
None anticipated — this separation is expected to remain correct even at
significantly higher scale, since it's driven by consistency/ownership
concerns rather than raw volume.
