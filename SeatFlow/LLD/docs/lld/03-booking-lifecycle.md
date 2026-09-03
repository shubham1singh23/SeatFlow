# 03 — Booking Lifecycle (State Machine)

## States

We use **five terminal-capable states plus two active states** — `CREATED`
is deliberately **not** a persisted state: creation and the initial hold
happen inside one DB transaction, so a booking is never observable in a
"created but not held" state. Persisting a state nothing can ever query is
dead weight.

| State | Meaning |
|---|---|
| `HELD` | Seats are claimed for this booking; TTL is running; no payment initiated yet |
| `PAYMENT_PENDING` | User initiated payment; Booking Service is waiting on Payment Service outcome |
| `CONFIRMED` | Payment succeeded; seats are permanently allocated to this booking |
| `CANCELLED` | User (or admin, where policy allows) cancelled the booking |
| `EXPIRED` | Hold TTL elapsed with no successful payment |
| `FAILED` | Payment attempt failed (declined, provider error) — booking may retry payment while still within hold TTL, or expire |

## Transition table

| From | Command | Guard | To | Side effects / events |
|---|---|---|---|---|
| — | `CreateBooking` (holds seats) | seats available (unique constraint holds); TTL assigned | `HELD` | insert `Booking`+`BookingItem`s in one txn; outbox `BookingHeld` |
| `HELD` | `InitiatePayment` | booking not expired; owner matches caller | `PAYMENT_PENDING` | call `PaymentServiceClient.initiate()` (outside DB txn — see `13-transaction-boundaries` doc) |
| `PAYMENT_PENDING` | `PaymentSucceeded` (webhook/callback) | idempotent on paymentId | `CONFIRMED` | mark items `CONFIRMED`; outbox `BookingConfirmed` |
| `PAYMENT_PENDING` | `PaymentFailed` | — | `FAILED` | outbox `BookingFailed`; items stay `HELD` if TTL not yet expired (see `10-payment-interaction.md` for retry policy) |
| `HELD` or `FAILED` | `ExpireHold` (background worker) | `now() > heldUntil` AND status not already terminal | `EXPIRED` | mark items `RELEASED`; outbox `BookingExpired` |
| `HELD` | `CancelBooking` (user) | owner matches caller | `CANCELLED` | mark items `RELEASED`; outbox `BookingCancelled` |
| `CONFIRMED` | `CancelBooking` (post-purchase, if policy allows) | owner matches caller; cancellation window open | `CANCELLED` | mark items `CANCELLED`; outbox `BookingCancelled` + triggers refund coordination via Payment Service (out of scope detail) |

### Explicitly invalid / rejected transitions
- `CONFIRMED → HELD` — never. Confirmation is a one-way door.
- `EXPIRED → CONFIRMED` — never via a normal path. The one case where
  payment succeeds after expiry is a distributed-systems edge case handled
  explicitly in `10-payment-interaction.md` ("late payment success"), and
  it does **not** silently flip status — it triggers a reconciliation
  action (refund), not a resurrection of the booking.
- Any transition attempted by a non-owner `userId` → rejected with 403,
  never silently ignored.
- Controllers **never** set status directly; only `BookingDomainService`
  applies transitions, and it validates the guard before mutating state
  (a simple internal state-transition table function, not a full State
  *pattern* — see `13-design-patterns.md` for why State pattern was
  considered and rejected as overkill for 7 states with no
  per-state polymorphic *behavior*, only transition legality).

## ASCII transition diagram

```
                 CreateBooking (seats claimed atomically)
                          |
                          v
                       [HELD] ------------------------------+
                       /   \  \                              |
      InitiatePayment /     \  CancelBooking     ExpireHold  | ExpireHold
                      v       v                              |
          [PAYMENT_PENDING] [CANCELLED]                      v
              /        \                                 [EXPIRED]
   PaymentSucceeded   PaymentFailed
             |              |
             v              v
        [CONFIRMED]     [FAILED] --(retry within TTL)--> [PAYMENT_PENDING]
             |                 \
     CancelBooking            ExpireHold (TTL elapsed)
             |                       |
             v                       v
        [CANCELLED]              [EXPIRED]
```
