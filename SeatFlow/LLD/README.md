# SeatFlow — Booking Service Low-Level Design

## What the Booking Service does
Booking Service owns the lifecycle of a *booking*: claiming seats for a
show (with an all-or-nothing, TTL-bound hold), coordinating payment via
Payment Service, confirming or expiring the booking, and publishing
booking-state events for Notification/Analytics to consume. It is the
system of record for **who currently holds or owns a seat for a show**.

## What it owns / does not own
- **Owns**: booking lifecycle & state machine, seat-claim orchestration,
  idempotency of booking commands, booking persistence & transaction
  boundaries, outbox events.
- **Does not own**: authentication (Auth Service), venue/seat/show catalog
  and pricing (Show Service), payment processing (Payment Service),
  notification delivery (Notification Service).

Full reasoning: `docs/lld/01-scope-and-responsibilities.md`.

## Core architectural decisions

1. **Concurrency (ADR-001, ADR-003)** — Postgres partial unique index
   (`UNIQUE(show_id, seat_id) WHERE status IN ('HELD','CONFIRMED')`) is the
   sole authority for the no-double-booking invariant. Redis `SET NX PX`
   is used only as a fail-open admission filter to cut DB load from
   doomed attempts on hot seats — never as the source of truth.
2. **Seat hold storage (ADR-002)** — no separate `SeatHold` table; a hold
   *is* a `BookingItem` in `HELD` status, since holds never exist
   independently of a booking in this system.
3. **Booking lifecycle (ADR-006)** — table-driven transition guards, not
   the State pattern (no per-state polymorphic behavior exists); no
   persisted `CREATED` state (creation and hold are atomic).
4. **Idempotency (ADR-004)** — Postgres-backed, using the same
   "let a unique constraint resolve the race" pattern as seat claiming.
5. **Outbox (ADR-005)** — transactional outbox for booking events;
   at-least-once delivery, consumer-side dedup on `event_id`; no claim of
   exactly-once.
6. **Payment coordination (ADR-007)** — async callback + reconciliation
   sweep, not a saga; `PAYMENT_PENDING` is excluded from hold-expiry
   eligibility so in-flight payments are never yanked out from under a
   user.

## Data ownership
Four tables: `bookings`, `booking_items` (carries hold fields —
see ADR-002), `idempotency_records`, `outbox_events`. Full schema:
`docs/lld/07-database-design.md`.

## Booking lifecycle
`HELD → PAYMENT_PENDING → CONFIRMED`, with `CANCELLED`/`EXPIRED`/`FAILED`
branches. Full state table: `docs/lld/03-booking-lifecycle.md`.

## Relation to the broader SeatFlow HLD
Booking Service sits between Show Service (catalog, read-only from here)
and Payment Service (coordinated, not owned) in the request path, and
publishes to Kafka for Notification/Analytics. It never talks to
Postgres/Redis on behalf of another service and never accepts direct
writes to seat/show catalog data.

## Major trade-offs
- Redis adds operational surface for a **latency**, not **correctness**,
  benefit — a deliberate, documented choice (ADR-001, `18-lld-tradeoffs.md`).
- At-least-once event delivery, not exactly-once — consumers must dedupe.
- No saga/CQRS/event sourcing — none are justified by a concrete
  requirement in this system today (`13-design-patterns.md`).

## Key interviewer discussion points
See the "Principal Engineer Review"-style questions embedded throughout
`docs/lld/` — most directly: `08-concurrency-and-seat-locking.md`,
`10-payment-interaction.md`, `11-event-and-outbox-design.md`, and
`18-lld-tradeoffs.md`.

---

## Folder structure

```
SeatFlow-Booking-Service-LLD/
├── README.md
├── docs/lld/              18 documents — scope through trade-offs
├── diagrams/lld/           9 editable .drawio diagrams + DIAGRAM_LINKS.md
├── decisions/              7 ADRs
└── research/               research-notes.md
```

## Diagram inventory

Each `.drawio` file is plain, human-readable mxGraph XML — open it
directly in the desktop/web Draw.io app (File → Import), or use the
one-click editable links in `diagrams/lld/DIAGRAM_LINKS.md` (each opens
straight into app.diagrams.net, no install, no account, diagram
pre-loaded). Links are kept in their own file because each one embeds the
full diagram XML and is several KB long.

| # | File | Purpose | What it communicates |
|---|---|---|---|
| 01 | `01-booking-domain-class-diagram.drawio` | Domain model | `Booking` aggregate, `BookingItem`, enums, value objects, and the cross-cutting `IdempotencyRecord`/`OutboxEvent` entities kept explicitly outside the aggregate |
| 02 | `02-booking-service-class-diagram.drawio` | Layered architecture | Controller → Application → Domain → Infrastructure, with correct (top-to-bottom) dependency direction |
| 03 | `03-concurrency-class-diagram.drawio` | Concurrency mechanism | Exactly which classes participate in enforcing the no-double-booking invariant, and in what order (Redis fast path, then Postgres authority) |
| 04 | `04-booking-database-schema.drawio` | Database schema | All 4 tables with PKs/FKs/unique index, and why `idempotency_records`/`outbox_events` have no FK to `bookings` |
| 05 | `05-create-booking-sequence.drawio` | Create Booking sequence | Full hold-seats flow, highlighting the single DB transaction that makes multi-seat acquisition all-or-nothing |
| 06 | `06-concurrent-booking-sequence.drawio` | Concurrent booking sequence | Two users, two service instances, one seat — shows exactly where (Postgres INSERT) the race is resolved |
| 07 | `07-payment-confirmation-sequence.drawio` | Payment confirmation sequence | Why the Payment Service call sits outside any DB transaction, and confirmation happens only in the callback |
| 08 | `08-payment-webhook-sequence.drawio` | Payment webhook sequence | Duplicate webhook delivery handled as a no-op via `paymentId` idempotency |
| 09 | `09-booking-expiration-sequence.drawio` | Booking expiration sequence | Background worker flow, the `PAYMENT_PENDING` exclusion guard, and the optimistic-lock race against a user action |

## Pattern inventory

| Pattern | Where used | Why |
|---|---|---|
| Repository | `BookingRepository`, `IdempotencyRecordRepository` | decouple domain from JPA |
| Adapter | `PaymentServiceClient`, `ShowServiceClient` | isolate external wire-format changes |
| Transactional Outbox | `OutboxService` + `OutboxRelay` | solve the DB/Kafka dual-write problem without XA |
| Builder | `BookingFactory` | readable, required-field-safe aggregate construction |

**Explicitly rejected** (with reasoning in `docs/lld/13-design-patterns.md`):
State Pattern, Saga orchestration, CQRS/Event Sourcing, Specification
Pattern, Strategy Pattern for pricing.

## Consistency verification performed

Before finalizing this package, the following cross-checks were
performed and passed:
- **Domain ↔ classes**: every domain concept in `02-domain-model.md` has
  exactly one corresponding class in `04-class-design.md`; no duplicated
  concepts (the rejected `SeatHold` table is the example of a concept
  deliberately *not* duplicated — ADR-002).
- **Classes ↔ database**: every persisted field in `04-class-design.md`'s
  entities has a matching column in `07-database-design.md`.
- **APIs ↔ state machine**: every endpoint in `06-api-design.md` maps to
  exactly one legal transition (or none, for reads) in
  `03-booking-lifecycle.md`'s table; no endpoint can perform a transition
  outside that table.
- **Concurrency ↔ database**: the mechanism described in
  `08-concurrency-and-seat-locking.md` is the literal DDL in
  `07-database-design.md` (`uq_active_seat_claim`), not a separate,
  unverified claim.
- **Redis ↔ Postgres**: every Redis-touching code path in
  `08-concurrency-and-seat-locking.md` and `12-error-and-failure-handling.md`
  fails open toward "ask Postgres," never toward "seat is free."
- **Payment ↔ booking state**: every payment outcome in
  `10-payment-interaction.md`'s table maps to exactly one transition in
  `03-booking-lifecycle.md`; the `PAYMENT_PENDING`-excluded-from-expiry
  rule appears identically in the lifecycle doc, the failure matrix, the
  expiration sequence diagram, and ADR-007.
- **Outbox ↔ events**: every state-changing transaction listed in
  `05-service-layer-design.md` includes an `OutboxService.record(...)`
  call in the same transaction.
- **Diagrams ↔ documentation ↔ classes**: all diagrams use the canonical
  names defined in `01-scope-and-responsibilities.md`
  (`BookingController`, `BookingApplicationService`,
  `BookingDomainService`, `SeatHoldDomainService`, `BookingRepository`,
  `SeatHoldCoordinator`, `IdempotencyService`, `OutboxService`,
  `OutboxRelay`, `PaymentServiceClient`, `ShowServiceClient`) with no
  synonyms introduced anywhere.
- **ADRs ↔ final design**: each ADR's "Decision" section matches the
  corresponding doc section it's paired with (ADR-001 ↔
  `08-concurrency-and-seat-locking.md`, ADR-005 ↔
  `11-event-and-outbox-design.md`, etc.) — no ADR contradicts the doc it
  documents.

## Honesty note

Everything in this package is **designed and reasoned**. Most of it is
**directly implementable** as specified (schema DDL, API contracts, class
responsibilities). The concurrency test scenario in
`docs/lld/16-testing-strategy.md` is specified precisely enough to
implement and run. **Nothing here has been benchmarked or run against
production traffic** — no performance numbers are claimed anywhere in
this package.
