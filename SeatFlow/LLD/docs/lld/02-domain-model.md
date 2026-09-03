# 02 — Domain Model

## Deliberate simplification: no separate `SeatHold` entity

The prompt (and most tutorials) suggest a standalone `SeatHold` concept.
We reject that here and explain why:

A "hold" and a "booking item" go through **the same lifecycle** in this
system: a seat is claimed (HELD) as part of one specific booking attempt,
and later becomes CONFIRMED, EXPIRED, RELEASED, or CANCELLED as part of
*that same* booking. There is no scenario in SeatFlow where a hold exists
independently of a booking it belongs to (we do not support "hold seats,
decide later which booking they attach to").

Therefore `BookingItem` **is** the hold: it carries `holdToken`,
`heldUntil` (TTL) and `status` in addition to seat/price data. Introducing
a second table (`seat_holds`) that mirrors `booking_items` 1:1 would be
duplicated state with a synchronization problem (two rows to keep
consistent) for zero additional expressive power. This is the kind of
"pattern for pattern's sake" the design explicitly avoids (see §35 in the
brief).

If SeatFlow later needed **holdable-without-a-booking** seats (e.g. a
"reserve for box office" feature), that would justify promoting the hold
back into its own aggregate — noted as a documented future evolution in
ADR-002.

## Aggregates, entities, value objects

### `Booking` (Aggregate Root)
- **Identity**: `bookingId` (UUID)
- **Fields**: `userId`, `showId`, `status` (`BookingStatus`), `items:
  List<BookingItem>`, `totalAmount` (`Money`), `createdAt`, `updatedAt`,
  `version` (optimistic locking for the aggregate row itself, distinct
  from the seat-claim concurrency mechanism — see `08-...md`)
- **Invariants**:
  - A `Booking` must have ≥1 `BookingItem`.
  - All `BookingItem`s belong to the same `showId`.
  - Status transitions only via the state machine (never set directly).
  - `totalAmount` is the sum of item prices, computed once at hold time
    and never silently recomputed (price integrity for the user).
- **Mutability**: mutable during HELD/PAYMENT_PENDING, then effectively
  frozen (append-only audit fields) once CONFIRMED/CANCELLED/EXPIRED/FAILED.

### `BookingItem` (Entity, child of `Booking`)
- **Identity**: `bookingItemId` (UUID), unique per (showId, seatId) among
  *active* rows (see DB design)
- **Fields**: `seatId`, `showId`, `priceAtHold` (`Money`), `status`
  (`BookingItemStatus`: HELD, CONFIRMED, RELEASED, CANCELLED),
  `holdToken`, `heldUntil`
- **Invariant**: enforced at the DB layer — for a given `(showId, seatId)`
  at most one `BookingItem` may be in an *active* status (`HELD` or
  `CONFIRMED`) at any time. This is the DB-level expression of the core
  system invariant.

### `IdempotencyRecord` (Entity, not part of the Booking aggregate)
- **Identity**: `idempotencyKey` (client-supplied, scoped to `userId` +
  operation type)
- **Fields**: `requestFingerprint` (hash of the request body), `status`
  (`NEW`, `IN_PROGRESS`, `COMPLETED`, `FAILED`), `responseSnapshot`
  (serialized result to replay), `expiresAt`
- Independent lifecycle from `Booking` — it can outlive or be cleaned up
  separately; it is a cross-cutting concern, not a domain concept of
  "booking."

### `OutboxEvent` (Entity, infrastructure-facing but written from the domain transaction)
- **Identity**: `eventId` (UUID, becomes the Kafka message key alongside
  aggregateId)
- **Fields**: `aggregateId` (bookingId), `eventType`, `payload` (JSON),
  `createdAt`, `publishedAt` (nullable), `retryCount`

## Value objects
- `Money` (amount + currency, immutable, arithmetic-safe — avoids float
  bugs)
- `HoldToken` (opaque string, proves possession of a hold when
  confirming/cancelling — prevents user B from confirming user A's hold
  even if they guessed a bookingId)

## Enums
- `BookingStatus`: `HELD`, `PAYMENT_PENDING`, `CONFIRMED`, `CANCELLED`,
  `EXPIRED`, `FAILED`
- `BookingItemStatus`: `HELD`, `CONFIRMED`, `RELEASED`, `CANCELLED`
- `PaymentStatus` (DTO mirror only, never persisted as owned state):
  `PENDING`, `SUCCEEDED`, `FAILED`

## Why "Venue Seat" vs "Show Seat" vs "Booking Item" are different things
- **Venue Seat** (Show Service): physical seat, exists once, belongs to a
  venue's layout. Booking Service never touches this.
- **Show Seat** (Show Service): a venue seat *in the context of one show*
  (carries the price for that show). Booking Service reads this via
  `ShowServiceClient` to validate existence/price — read-only, not stored
  long-term.
- **BookingItem** (Booking Service): the *claim* on a show seat by one
  specific booking. This is the only one Booking Service persists and is
  authoritative for.

Collapsing "Show Seat" and "BookingItem" would force Booking Service to
own pricing/catalog data it has no business owning and would need to keep
in sync with Show Service. Keeping them separate keeps each service's
source of truth unambiguous.
