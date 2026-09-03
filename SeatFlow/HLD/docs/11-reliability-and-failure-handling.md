# 11 — Reliability & Failure Handling

## Comprehensive Failure Matrix

| Failure | Impact | Detection | Recovery | Consistency Implication |
|---|---|---|---|---|
| **Redis unavailable** | Loss of fast-path admission filter for holds and read-through cache | Connection errors / circuit breaker trip | Fallback to Postgres-only conditional `UPDATE` for holds (slower under contention); Catalog reads fall through to Postgres replica | **None** — correctness backstop is Postgres, always. Degrades throughput only. |
| **PostgreSQL (Booking DB) unavailable** | Cannot create/confirm bookings | Connection pool exhaustion / health check failure | Circuit breaker returns 503 for writes; reads may still work via replica; automatic recovery once DB is back; outbox events queued once DB returns | Deliberate **availability sacrificed for consistency** on the write path — no booking is ever accepted without durable guarantee capability |
| **Kafka unavailable** | Outbox events accumulate unpublished (`published=false`); no immediate downstream side effects (notifications, cross-service confirmation) | Outbox publisher error metrics, growing unpublished backlog | Outbox publisher retries with backoff; once Kafka recovers, backlog drains, at-least-once delivery resumes | Business-critical writes (booking, payment rows) already committed to Postgres before Kafka is touched — **no data loss**, only delayed downstream effects |
| **Booking Service instance crash** | In-flight requests on that instance fail (client sees timeout/5xx) | Load balancer health check | LB routes around the dead instance; client retries with same `Idempotency-Key` → safe replay | None — stateless service, idempotency key protects against duplicate side effects |
| **Payment Service instance crash** | Same as above for payment intent creation / webhook processing | Health check | LB reroutes; webhook providers retry failed webhook deliveries automatically (standard provider behavior) | None — webhook idempotency via `eventId` |
| **Notification Service down/slow** | Delayed or missing notifications | Consumer lag metric | Independent consumer group scales/recovers on its own schedule; DLQ + replay for poison messages | None to booking/payment — fully isolated blast radius by design |
| **Network timeout (client ↔ gateway ↔ service)** | Client doesn't know if the operation succeeded | Client-side timeout | Client retries with the **same** `Idempotency-Key`; server returns the original result rather than re-executing | None — idempotency key is the mechanism |
| **Duplicate HTTP request (client double-click, proxy retry)** | Same request arrives twice | `Idempotency-Key` lookup finds existing record | Second request short-circuits to stored result | None |
| **Duplicate Kafka event (at-least-once redelivery)** | Consumer sees the same event twice | Consumer checks domain state before acting (state-machine guard) | No-op on redelivery | None |
| **Duplicate payment webhook** | Provider redelivers | Unique constraint on `eventId` | Upsert is a no-op | None |
| **Payment succeeds + booking confirm fails (transient)** | Booking briefly stuck `PENDING_PAYMENT` despite paid | `PaymentSucceeded` event delivery is at-least-once via outbox | Consumer retries/redelivers until booking is confirmed; reconciliation job as final backstop | Eventually resolved automatically; never silently lost, because the event was durably recorded in the same transaction as the payment success |
| **Booking succeeds + notification fails** | User isn't emailed promptly | Notification consumer error/DLQ metrics | Retry/DLQ replay within Notification Service | None to core business state |
| **Application crashes mid seat-hold** (between Redis SET and Postgres UPDATE) | Redis key exists, no Postgres HELD row | TTL-based | Redis key expires in ≤300s, seat becomes bookable again | Temporary availability cost only, never a correctness violation (see `07-concurrency-design.md`) |
| **User abandons checkout** | Hold never converted to booking | Hold `expires_at` reached | Sweeper job releases the seat back to AVAILABLE | None |

## Guiding Principle

> Every failure mode above resolves to one of: **(a) safe automatic retry
> via idempotency**, **(b) safe eventual consistency via at-least-once
> events + idempotent consumers**, or **(c) a deliberate, bounded
> availability trade-off in exchange for never violating the seat/payment
> correctness guarantee.**

No failure mode in this system is handled by "hope it doesn't happen."

## Circuit Breakers & Timeouts

Synchronous inter-service calls (Booking → Payment) use bounded timeouts
and a circuit breaker (open after a failure-rate threshold, half-open
probe, close on recovery) so a degraded downstream service produces fast,
clear failures upstream instead of thread/connection exhaustion cascading
across the system.

## Graceful Degradation Summary

| Under this outage… | …this stays up | …this is degraded/down |
|---|---|---|
| Redis down | Everything (via Postgres fallback) | Hold throughput under hot-seat contention |
| Kafka down | Booking, Payment, browsing | Notifications, cross-service confirmation latency |
| Payment provider down | Browsing, existing bookings | New payments |
| Notification provider down | Everything else | Notifications only |
| Booking DB down | Browsing (cached/replica), existing booking reads | New holds/bookings — deliberately |
