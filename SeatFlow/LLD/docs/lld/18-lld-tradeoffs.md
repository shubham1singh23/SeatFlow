# 18 — LLD Trade-offs & SOLID/OOP Review

## SOLID review

- **SRP**: each class in `04-class-design.md` has one reason to change —
  verified by asking "what would force this class to change?" for each
  (e.g. `SeatHoldCoordinator` only changes if the fast-path claim
  mechanism changes, never if a state-machine rule changes).
- **OCP**: payment outcome handling is a single method with an explicit
  switch on `outcome` today — if a new payment provider needs different
  handling, `PaymentServiceClient` is the seam (an interface), not a
  change to `BookingDomainService`. Booking policy (max seats, hold TTL)
  is externalized as configuration, not hardcoded, so it can change
  without a redeploy of logic.
- **LSP**: `PaymentServiceClient`/`ShowServiceClient` are interfaces with
  one real implementation today; a test double substitutes cleanly
  because the interface only exposes what Booking Service actually needs
  (see ISP below), not the full external API surface.
- **ISP**: `ShowServiceClient` exposes only `validateSeats(showId,
  seatIds)` and `getSeatPrice(...)` — not the entire Show Service API
  surface. Booking Service never depends on methods it doesn't call.
- **DIP**: `BookingDomainService`/`SeatHoldDomainService` depend on
  repository/client **interfaces**, injected — never on JPA or WebClient
  concrete types directly.

## Is `BookingApplicationService` a God class?
Reviewed explicitly: it has 3 public methods, each a thin
orchestration (idempotency → domain call → outbox → response), with all
actual logic delegated. If it grows beyond orchestration (e.g. starts
containing seat-claim retry loops or payment-outcome branching logic
inline), that's the signal to extract — documented here so a future
reviewer has a concrete trigger, not a vague "watch out for growth."

## Why not CQRS / Event Sourcing / Saga / distributed transactions
See `13-design-patterns.md`'s rejection list — each was evaluated against
a concrete requirement in this document and found to solve a problem
SeatFlow's Booking Service does not currently have. Adding them now would
be complexity spent on hypothetical future requirements rather than the
concurrency/consistency requirements that are actually in front of us —
directly against §35 of the brief ("do not overengineer").

## Biggest trade-off in the whole design
The hybrid Redis+Postgres concurrency approach (`08-...md`, Option G)
accepts **extra operational surface** (two systems, not one, involved in
the hot path) in exchange for **latency under contention** without ever
weakening the correctness guarantee, because the guarantee is fully
carried by Postgres alone (Option F) and Redis is strictly additive. If
this were a lower-traffic system where popular-show contention wasn't a
real concern, Option F alone would be the simpler, equally-correct choice
— noted explicitly as a valid simpler alternative in ADR-001.

## Honesty about status
Everything in this LLD is **designed and reasoned**, most of it is
**directly implementable** as specified, some of it (the concurrency test
scenario in `16-testing-strategy.md`) is **specified precisely enough to
implement and run**, but nothing here has actually been benchmarked or run
against production traffic — no performance numbers are claimed anywhere
in this package.
