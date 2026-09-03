# 05 — Service Layer Design & Transaction Boundaries

## Principle
`@Transactional` marks exactly the span of work that must be atomic **and
DB-only**. Any call to another service (Payment, Show) or to Redis is
never inside the same transaction as a DB write, because a slow/failed
remote call must not hold Postgres locks or a long-lived transaction open.

## Per-operation breakdown

### Create booking (hold seats)
```
BEGIN TRANSACTION (BookingApplicationService.createBooking)
  IdempotencyService.begin(key)          -- DB write
  ShowServiceClient.validateSeats(...)   -- OUTSIDE txn, called BEFORE begin,
                                             result passed in as validated data
  SeatHoldCoordinator.tryClaim(...)      -- Redis, OUTSIDE the DB txn (called
                                             just before, fast, non-transactional)
  INSERT booking, booking_items          -- DB write (unique index enforces invariant)
  OutboxService.record(BookingHeld)      -- DB write, same txn
COMMIT
IdempotencyService.complete(key, response)   -- can be same txn or immediately after
```
Rolls back on: unique-constraint violation (seat taken), validation
failure. Cannot roll back: the Redis claim already made — handled by a
short Redis TTL so a rolled-back claim self-heals within seconds even if
explicit release is skipped (defense in depth, not the primary mechanism).

### Confirm booking (payment callback)
```
BEGIN TRANSACTION
  Look up booking by paymentId/bookingId, check status guard
  UPDATE booking status=CONFIRMED, booking_items status=CONFIRMED
  OutboxService.record(BookingConfirmed)
COMMIT
```
The call *to* Payment Service (`initiatePayment`) happens **before** this
transaction, in `PAYMENT_PENDING`-setting step, never inside it — payment
provider latency must never hold a DB transaction open.

### Cancel booking
Single transaction: guard check → status update → release (unique index
predicate no longer matches) → outbox record. No external calls needed
inside the transaction.

### Expire booking (background worker)
Single transaction per booking (batched query, per-row transaction to
avoid one huge lock scope): guard (`status IN HELD/FAILED AND held_until <
now()`) → update → outbox record. Uses the row's `version` for optimistic
concurrency against a racing user action (e.g. user confirms in the same
instant the worker tries to expire) — whichever transaction commits first
wins; the loser's optimistic-lock failure is treated as "no-op, already
handled."

### Process payment callback
Guard: only apply if booking is currently `PAYMENT_PENDING` **or** the
`paymentId` was already applied (idempotent no-op) — see
`10-payment-interaction.md`. Single transaction, no external calls inside
it.

### Create outbox event
Never its own transaction — always joins the caller's transaction
(`OutboxService.record` uses `Propagation.MANDATORY` to make this
explicit and fail loudly if someone calls it outside a transaction).

## Why not `@Transactional` on `BookingController` or on every method
Putting it on the controller would wrap external HTTP calls to Show/
Payment Service inside a DB transaction — exactly the anti-pattern this
design avoids. Putting it on every method (including pure read methods
with no writes) adds needless connection-pool pressure. `@Transactional`
appears only on the four write orchestration methods listed above.
