# 15 — Observability

## Logging
Structured (JSON) logs carrying: `bookingId`, `userId`, `showId`,
`requestId`, `correlationId` (propagated from the Gateway across Booking →
Payment/Show calls), `idempotencyKeyHash` (hashed, never the raw key if
it could be considered sensitive), `eventId` (on outbox-related logs).
**Never logged**: `paymentMethodToken`, any payment provider response
body, raw card-adjacent data.

## Metrics (chosen for actionability, not exhaustiveness)
```
booking_attempts_total{result=success|seat_unavailable|validation_error}
seat_hold_conflict_total          -- direct signal of contention hot-spots
booking_confirmation_latency_seconds   -- HELD -> CONFIRMED, payment included
payment_dependency_latency_seconds
outbox_publish_lag_seconds        -- age of oldest unpublished row
outbox_publish_failure_total
booking_expiry_worker_runs_total{holds_expired}
redis_fastpath_hit_ratio          -- signals whether Redis is earning its keep
```
`seat_hold_conflict_total` and `redis_fastpath_hit_ratio` are the two
metrics that would tell an on-call engineer, during a blockbuster on-sale,
whether the hybrid concurrency design (`08-...md`) is behaving as
designed.

## Tracing
`correlationId` generated at the Gateway, propagated as a header through
Booking Service → `PaymentServiceClient`/`ShowServiceClient` calls, and
included in the outbox event payload so Notification/Analytics can
continue the same trace across the Kafka boundary. A single distributed
trace should be reconstructable end-to-end:
`Gateway → BookingController → BookingApplicationService →
PaymentServiceClient → (Payment Service) → payment-callback → Kafka
(via correlationId in event payload) → Notification Service`.
