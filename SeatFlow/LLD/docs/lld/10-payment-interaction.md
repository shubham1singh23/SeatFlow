# 10 — Payment Service Interaction

Booking Service never talks to a payment provider. It calls
`PaymentServiceClient.initiatePayment(bookingId, amount, idempotencyKey)`
and later receives an outcome either via a synchronous poll/response or
(preferably, and what this design assumes) an async **callback/webhook
from Payment Service → Booking Service** (`POST
/internal/bookings/{id}/payment-callback`), itself idempotent on
`paymentId`.

## Case-by-case analysis

| Case | Behavior |
|---|---|
| Payment succeeds, callback arrives normally | `PAYMENT_PENDING → CONFIRMED`, idempotent on `paymentId` (second identical callback is a no-op, see below) |
| Payment fails | `PAYMENT_PENDING → FAILED`. If `held_until` not yet passed, booking stays claimable for a retry (`InitiatePayment` again); otherwise the expiry worker later moves it to `EXPIRED` |
| Payment times out (Booking Service → Payment Service call itself times out) | Booking Service does **not** assume failure. It leaves status at `PAYMENT_PENDING` and relies on the callback (push) as the source of truth, with a reconciliation poll as a backstop (`PaymentServiceClient.getStatus(paymentId)` on a scheduled sweep for stuck `PAYMENT_PENDING` bookings older than N minutes) |
| Payment succeeds but the response to Booking Service is lost (network drop after Payment Service committed) | Recovered by the reconciliation sweep above — it asks Payment Service for ground truth rather than assuming |
| Booking Service crashes after calling Payment Service, before persisting anything | On restart, the booking is still `HELD` (nothing was persisted) or `PAYMENT_PENDING` if that write landed. Either way the reconciliation sweep resolves it against Payment Service's ground truth — Booking Service never *invents* an outcome |
| User retries `InitiatePayment` | Idempotency key on the payment-initiation request (per `09-idempotency.md`) — Payment Service is expected to also dedupe on this key; Booking Service does not initiate two payments for one booking |
| Webhook/callback arrives twice | `PaymentCallbackConsumer` looks up `paymentId` in `booking_items`/`bookings` audit trail; second callback with the same `paymentId` and an already-`CONFIRMED` booking is a **no-op 200** (idempotent, not an error) |
| Webhook arrives before the "expected" synchronous response | Treated as the authoritative outcome the moment it arrives — Booking Service does not require the synchronous call to have "finished" first; the callback handler is independent and idempotent |
| Hold expires while payment is processing | The expiry worker only expires bookings whose status is `HELD` or `FAILED` (see `03-booking-lifecycle.md`'s guard) — `PAYMENT_PENDING` is explicitly excluded from expiry eligibility, so a payment genuinely in flight is never yanked out from under the user |
| **Payment succeeds *after* the seat hold TTL already expired** (edge case) | Can only happen if the expiry worker incorrectly expired a `PAYMENT_PENDING` booking (should not happen per the guard above) — but as a defense-in-depth: if a late `PaymentSucceeded` callback targets a booking already `EXPIRED`, Booking Service does **not** silently flip it back to `CONFIRMED`. It marks the booking `CONFIRMED` **only if the seat is still uncontested** (re-checks the unique constraint); if another booking has since claimed the seat, it records the conflict and emits a `LatePaymentConflict` event for manual/automated refund — correctness of the seat invariant always wins over payment convenience |

## Why the callback path, not synchronous blocking

Payment provider round-trips (3-D Secure, bank redirects) can take
seconds to minutes — holding an HTTP request or a DB transaction open for
that long is not viable. `PAYMENT_PENDING` is exactly the state that lets
Booking Service step out of the critical path and resume when Payment
Service tells it the outcome, sync or async.
