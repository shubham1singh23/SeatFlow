# 10 — Payment Architecture

## The Dual-Write Problem & The Outbox Pattern

Both Booking Service and Payment Service face the same hazard: "update my
database" and "publish a Kafka event" are two separate operations. If the
DB commit succeeds and the Kafka publish fails (network blip, broker
unavailable), the rest of the system never learns about a state change
that definitely happened — a silent, hard-to-detect inconsistency.

**Decision: Transactional Outbox**, in both Booking and Payment services.

```
BEGIN TRANSACTION
   UPDATE payments SET status = 'SUCCEEDED' WHERE id = ?
   INSERT INTO payment_outbox (event_type, payload) VALUES ('PaymentSucceeded', ...)
COMMIT
        │
        ▼
Outbox Publisher (poll or CDC) reads unpublished rows,
publishes to Kafka, marks published = true
```

The business-state change and the "I owe the world this event" record
commit **atomically in the same local transaction** — there is no window
where one exists without the other. The publisher then guarantees
**at-least-once** delivery to Kafka (retries on publish failure; a row
stays `published=false` until a publish is acknowledged). Consumers are
idempotent (see `09-event-driven-architecture.md`), so at-least-once
delivery from the outbox is safe.

Rejected alternative: **dual-write without outbox** (write DB, then
publish directly). Simpler, but reintroduces exactly the failure mode
above — rejected because payment/booking state divergence is precisely
the class of bug this whole project is designed to eliminate.

## End-to-End Payment Flow

```
Client                Booking Svc         Payment Svc      Payment Provider
  │                        │                    │                  │
  │ POST /bookings         │                    │                  │
  │ (holdId, Idem-Key)     │                    │                  │
  ├───────────────────────►│                    │                  │
  │                        │ create booking      │                  │
  │                        │ (PENDING_PAYMENT)   │                  │
  │                        │ POST /payments/intents                │
  │                        ├───────────────────►│                  │
  │                        │                    │ create intent w/ │
  │                        │                    │ provider, store  │
  │                        │                    │ locally          │
  │                        │                    ├─────────────────►│
  │                        │◄───────────────────┤                  │
  │◄───────────────────────┤ paymentIntentId    │                  │
  │  redirect / client SDK for provider checkout                   │
  ├─────────────────────────────────────────────────────────────► │
  │                                                    user pays   │
  │                                                                │
  │                                          webhook (payment result)
  │                                          ◄─────────────────────┤
  │                                Payment Svc: verify signature,  │
  │                                idempotent upsert on providerEventId,
  │                                update payments row, write outbox row
  │                                          │                     │
  │                            (async) PaymentSucceeded event via Kafka
  │                                          ▼                     │
  │                                   Booking Svc consumer:        │
  │                                   confirm booking (ACID tx,    │
  │                                   unique-index-protected)      │
  │                                          │                     │
  │                            BookingConfirmed event → Notification
```

## Idempotency in Payments

| Operation | Idempotency mechanism |
|---|---|
| `POST /payments/intents` | Client `Idempotency-Key` header → stored result returned on retry, no duplicate provider charge intent created |
| Webhook delivery | Provider's `eventId` is unique-constrained locally; a redelivered webhook (providers retry until 2xx) is a no-op upsert, not reprocessed |
| Booking confirmation from `PaymentSucceeded` | Consumer checks `booking.status != PENDING_PAYMENT` before acting (see event-driven doc) |

## Failure Scenarios Specific to Payments

| Scenario | Handling |
|---|---|
| **Payment succeeds, but app/Booking Service crashes before confirming booking** | Payment state is durable in Payment DB regardless. `PaymentSucceeded` event was already durably written to the outbox in the same transaction as the payment update, so it **will** be delivered (at-least-once) once Booking Service recovers — the confirm-on-consume logic is idempotent, so it confirms correctly whenever it eventually processes the event. Nothing is lost; worst case is a delay, not an incorrect state. |
| **Booking succeeds, notification fails** | Isolated to Notification Service; booking/payment state is already durably correct. Notification retries independently (its own consumer offset / DLQ). |
| **Duplicate webhook (provider retries)** | Unique constraint on provider `eventId` in `payments`/a dedicated `webhook_events` table makes reprocessing a no-op. |
| **Payment timeout (provider never responds)** | Payment intent has its own expiry; if no webhook arrives within a bounded window, a reconciliation job actively queries the provider's status API (never assumes failure or success from silence alone) and resolves the intent, which also allows the linked hold to expire naturally if unresolved. |
| **User abandons checkout entirely** | No payment ever initiated or it stays unresolved; the seat hold's independent TTL (`07-concurrency-design.md`) releases the seat regardless of payment outcome. |

## Reconciliation

A scheduled reconciliation job periodically compares Payment Service's
local `PENDING`-state intents older than a threshold against the
provider's status API, and resolves any that the webhook mechanism missed
(webhooks are best-effort delivery even from reputable providers — never
the *only* mechanism trusted for a stuck payment).
