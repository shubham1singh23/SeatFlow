# 04 — Class Design

```
booking/
├── api/            BookingController, dto/, mapper/
├── application/    BookingApplicationService
├── domain/
│   ├── model/       Booking, BookingItem, enums, value objects
│   ├── service/      BookingDomainService, SeatHoldDomainService
│   ├── event/        BookingHeld, BookingConfirmed, ... (event payload records)
│   └── exception/     SeatUnavailableException, InvalidTransitionException, ...
├── infrastructure/
│   ├── persistence/   BookingRepository (JPA), BookingItemRepository
│   ├── redis/          SeatHoldCoordinator
│   ├── outbox/          OutboxService, OutboxRelay
│   ├── idempotency/     IdempotencyService
│   └── clients/          PaymentServiceClient, ShowServiceClient
└── configuration/
```

## Class responsibilities

**`BookingController`** (api layer)
- Maps HTTP ↔ DTOs, extracts `userId` from the authenticated principal,
  extracts `Idempotency-Key` header.
- No business logic. Delegates every operation to
  `BookingApplicationService`. Translates domain exceptions to HTTP status
  via a `@ControllerAdvice`.

**`BookingApplicationService`** (application layer — orchestration)
- Public methods: `createBooking(cmd)`, `confirmBooking(id, cmd)`,
  `cancelBooking(id, cmd)`.
- Responsibility: orchestrate idempotency check → domain call → outbox →
  response assembly. Owns transaction boundaries (`@Transactional`
  methods live here, never in the controller — see `05-service-layer-design.md`).
- Depends on: `IdempotencyService`, `BookingDomainService`,
  `SeatHoldDomainService`, `OutboxService`, `PaymentServiceClient`,
  `ShowServiceClient`.

**`BookingDomainService`** (domain layer)
- Pure business logic: validates state transitions against the table in
  `03-booking-lifecycle.md`, computes `totalAmount`, builds `Booking`/
  `BookingItem` aggregates. **No Spring annotations, no I/O** — fully
  unit-testable without mocks beyond plain objects.

**`SeatHoldDomainService`** (domain layer)
- Encapsulates the "claim these seats, all-or-nothing" algorithm described
  in `08-concurrency-and-seat-locking.md`: calls `SeatHoldCoordinator`
  (Redis fast path) then delegates the authoritative insert to
  `BookingRepository`, translating a unique-constraint violation into
  `SeatUnavailableException`.

**`BookingRepository`** (Spring Data JPA)
- Standard CRUD + `findExpiredHolds(now)` query backing the expiry worker.
- Not extended with locking annotations — correctness is index-driven, not
  lock-driven (see `08-...md`).

**`SeatHoldCoordinator`** (infrastructure/redis)
- `tryClaim(showId, seatId, holdToken, ttl): boolean` — thin wrapper over
  `SET key NX PX ttl`. `release(showId, seatId)` — `DEL` on
  confirm/cancel/expire. Fails safe: any Redis exception is swallowed and
  treated as "not claimed by fast path, fall through to DB" — see
  `12-error-and-failure-handling.md`.

**`IdempotencyService`** — see `09-idempotency.md`.

**`OutboxService`** — `record(bookingId, eventType, payload)` — writes the
outbox row inside the caller's existing transaction (no `@Transactional`
of its own — it must join whatever transaction is already open).

**`OutboxRelay`** — `@Scheduled` poller, no HTTP entry point, publishes to
Kafka and marks rows published. See `11-event-and-outbox-design.md`.

**`PaymentServiceClient` / `ShowServiceClient`** — thin adapters
(`WebClient`/Feign-style) around the external services' contracts;
translate their DTOs into internal value objects so domain code never
depends on another service's wire format.

## Avoiding a God class
`BookingApplicationService` is the only class every request passes
through, but it *delegates* — it contains no seat-claim algorithm, no
state-machine rule, no persistence detail, no Redis knowledge. Each of
those lives in exactly one other class. If `BookingApplicationService`
ever grows validation/business rules inline, that is the signal to extract
them — reviewed explicitly in `20-solid-review` considerations captured in
`18-lld-tradeoffs.md`.
