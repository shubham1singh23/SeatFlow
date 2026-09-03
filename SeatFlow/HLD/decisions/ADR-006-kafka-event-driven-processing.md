# ADR-006: Kafka for Cross-Service Side Effects (Not for the Hold Decision)

## Context
Several state changes (payment success, booking confirmation) have
side effects in other services (notification, cross-service confirmation)
that must not block the triggering request and must not be lost if the
consuming service is briefly down.

## Problem
How should Booking/Payment communicate state changes to interested
services without coupling their availability together?

## Options Considered

### Option 1: Synchronous REST calls to every interested service
Simple, but couples the caller's success to every callee's availability —
exactly the coupling ADR-001 exists to avoid (Notification down would
threaten Payment's ability to complete a webhook response).

### Option 2: Kafka, at-least-once, with idempotent consumers
Decouples producer availability from consumer availability entirely;
durable, replayable.

### Option 3: No async layer — poll for state changes
Would work but adds latency and unnecessary DB load for something an
event system does more naturally.

## Decision
Option 2 — Kafka for `BookingCreated`, `PaymentSucceeded`, `PaymentFailed`,
`BookingConfirmed`, `BookingCancelled`, `RefundInitiated/Completed`. The
hold decision itself (`07-concurrency-design.md`) remains synchronous
Redis+Postgres — Kafka is deliberately **not** used there, since the user
is actively waiting on that specific answer within the same request.

## Why We Chose It
At-least-once delivery + idempotent consumers gives durability without
requiring distributed transactions across service boundaries — the
correctness of the *business* state (in each service's own Postgres) is
never dependent on Kafka being available at the moment of the original
write (see ADR-007, Outbox Pattern).

## Trade-offs
- Introduces eventual consistency for confirmation/notification — a
  booking can be briefly `PENDING_PAYMENT` even after payment has
  succeeded, until the event is consumed. Accepted per the stated
  consistency boundary in `01-requirements.md`.
- Requires every consumer to be written idempotently — a real
  engineering discipline requirement, not free.

## Consequences
- Partition key choices (`show_id` for booking-events, `booking_id` for
  payment-events) preserve ordering where it matters without requiring
  global ordering.
- DLQ per topic needed so a poison message can't stall a partition
  indefinitely.

## Failure Scenarios
See `11-reliability-and-failure-handling.md` (Kafka unavailable row) and
`09-event-driven-architecture.md` (retry/DLQ flow).

## Future Evolution
Analytics/reporting consumers could be added on these same topics without
any change to producers — a natural benefit of the event-driven boundary
already being in place.
