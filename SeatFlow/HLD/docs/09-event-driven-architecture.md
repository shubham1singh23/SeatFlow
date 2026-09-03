# 09 — Event-Driven Architecture

## Why Kafka, and Where

Kafka is used **only** where a state change has side effects that (a) the
triggering request should not have to wait on, and (b) must not be lost
even if the immediate consumer is temporarily down. It is not used as a
general RPC replacement — see `03-system-architecture.md` for the
synchronous-vs-asynchronous boundary.

## Events

| Event | Producer | Consumer(s) | Why async |
|---|---|---|---|
| `BookingCreated` | Booking Service | Notification Service | User shouldn't wait on notification dispatch |
| `PaymentInitiated` | Payment Service | (analytics — future) | Not on critical path |
| `PaymentSucceeded` | Payment Service | Booking Service, Notification Service | Booking confirmation triggered by payment result, which itself arrives asynchronously via webhook |
| `PaymentFailed` | Payment Service | Booking Service, Notification Service | Release the hold, notify user |
| `BookingConfirmed` | Booking Service | Notification Service | Send confirmation |
| `BookingCancelled` | Booking Service | Payment Service (trigger refund), Notification Service | Cancellation and refund are decoupled steps |
| `RefundInitiated` / `RefundCompleted` | Payment Service | Notification Service | Refund settlement is provider-async by nature |

## Topics, Partitioning, Ordering

| Topic | Partition key | Why |
|---|---|---|
| `booking-events` | `show_id` | All events for a given show are ordered relative to each other — useful for reasoning about a show's inventory timeline; spreads load across shows |
| `payment-events` | `booking_id` | All events for one booking's payment lifecycle stay ordered |

Ordering is **not** relied upon across topics/keys — Booking Service
consumers are written to be correct regardless of `PaymentSucceeded` vs a
duplicate/late `PaymentInitiated` arriving in any order, because...

## Idempotent Consumers

Every consumer keys off the event's **domain ID** (e.g. `paymentId`,
`bookingId`) and treats handling as an *upsert to a state machine*, not an
imperative action:

```
on PaymentSucceeded(bookingId, paymentId):
    if booking.status != 'PENDING_PAYMENT':
        return   -- already handled (duplicate delivery, replay, retry)
    confirmBooking(bookingId)   -- single ACID transaction
```

This makes replays, redeliveries, and Kafka's **at-least-once** delivery
semantics safe by construction, without needing exactly-once transactions
across the whole pipeline.

## Consumer Groups & Scaling

Each logical consumer (Booking's payment-event handler, Notification
Service) runs as its own **consumer group**, scaled by adding instances up
to the partition count of the topic it reads. Booking's hot path
(confirm/release on payment result) and Notification's dispatch are
independent consumer groups so a slow Notification consumer can never
create backpressure on booking confirmation.

## Retries & Dead-Letter Queue

```
Consumer processes event
        │
     success ──► commit offset
        │
     failure (transient: DB timeout, etc.)
        │
   retry with backoff (bounded, e.g. 3 attempts, in-process)
        │
   still failing?
        │
   publish to {topic}-dlq, commit original offset (don't block partition)
        │
   alert + manual/automated replay tooling
```

A `payment-events-dlq` / `booking-events-dlq` exists per topic. DLQ
messages retain full context (original event + failure reason) so they can
be inspected and replayed after the underlying issue (e.g. a Booking DB
outage) is resolved. A poison-pill message can never permanently stall a
partition, because failed messages are shunted to the DLQ rather than
retried forever in place.

## Why Not Kafka for the Hold Decision Itself

The seat-hold decision (`07-concurrency-design.md`) is deliberately
**synchronous** (Redis + Postgres), not event-driven — the user is
actively waiting on that specific answer within the same HTTP request.
Kafka is reserved for genuine "and then, separately, this should also
happen" work.
