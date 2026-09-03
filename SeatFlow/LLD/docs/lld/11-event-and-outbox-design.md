# 11 — Event & Outbox Design

## The dual-write problem

Naively: `UPDATE bookings SET status='CONFIRMED'` then `kafkaProducer.send(...)`.
If the process crashes between the two, or the Kafka send fails, the DB
says CONFIRMED but Notification/Analytics never hear about it — silent
inconsistency with no way to detect it after the fact.

## Solution: Transactional Outbox

```
One DB transaction:
  UPDATE bookings SET status = 'CONFIRMED', ...
  INSERT INTO outbox_events (event_id, aggregate_id, event_type, payload, ...)
COMMIT   <-- both happen or neither happens, standard ACID guarantee
```

`OutboxRelay` (a separate scheduled component, polling
`idx_outbox_unpublished`) reads unpublished rows, publishes to Kafka, and
marks `published_at`. It does **not** run inside the booking transaction —
Kafka availability must never block a booking decision.

## Ordering & delivery semantics

- Kafka partition key = `aggregate_id` (bookingId), so all events for one
  booking are strictly ordered within their partition.
- Delivery is **at-least-once**, not exactly-once. If `OutboxRelay`
  crashes after a successful Kafka publish but before writing
  `published_at`, the same row is republished on the next sweep. This is
  a real, acknowledged trade-off — not glossed over.
- **Consumer idempotency is therefore a consumer-side responsibility.**
  `event_id` is included in every payload specifically so Notification
  Service / Analytics Service can dedupe (e.g. an idempotency table on
  their side, or a Kafka Streams dedup store) — Booking Service documents
  this contract but cannot enforce a consumer's internal implementation.
- Retry: failed publishes increment `retry_count` and are retried with
  backoff on the next sweep; after a configurable max (`retry_count > N`)
  the row is flagged for a dead-letter alert rather than retried forever
  (prevents one poison event from starving the relay).

## Outbox schema
See `07-database-design.md#outbox_events`. `payload` is the fully-formed
event DTO (bookingId, status, seatIds, userId, showId, timestamp) so
consumers never need to call back into Booking Service to enrich the
event.

## What we do NOT claim
We do **not** claim exactly-once delivery. We claim: **every state change
that was durably committed will eventually be published at least once**,
and consumers are responsible for deduplicating on `event_id`. This is the
honest, standard trade-off (see Chris Richardson's microservices.io outbox
pattern write-up, referenced in `research/research-notes.md`).
