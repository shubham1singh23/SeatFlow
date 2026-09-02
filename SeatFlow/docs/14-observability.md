# 14 — Observability

## Structured Logging & Correlation

Every request is tagged with a `correlationId` at the API Gateway
(generated if not supplied) and propagated through every downstream
synchronous call (header) and every published Kafka event (payload
metadata). Logs are structured JSON (`timestamp, service, correlationId,
userId?, event, ...`), enabling a single seat-hold-to-confirmation flow to
be reconstructed across five services and two async hops from one
`correlationId`.

## Distributed Tracing

OpenTelemetry instrumentation across all services, exported to a tracing
backend (e.g. Jaeger). Spans cover: gateway → service, service → Postgres,
service → Redis, service → Kafka produce/consume, service → external
provider. This is what makes "why did this specific booking take 800ms"
answerable rather than guessed at.

## Metrics (Micrometer → Prometheus → Grafana)

### Business metrics

```
booking_hold_total{result="success|conflict"}
booking_hold_expired_total
booking_confirmed_total
booking_cancelled_total
payment_success_total
payment_failure_total
webhook_duplicate_total
```

### Technical metrics

```
booking_hold_latency_seconds        (histogram)
booking_confirm_latency_seconds     (histogram)
payment_latency_seconds             (histogram)
redis_setnx_failure_rate
db_connection_pool_utilization
kafka_consumer_lag{topic, group}
outbox_unpublished_backlog{service}
circuit_breaker_state{dependency}
```

`booking_hold_total{result="conflict"}` and `redis_setnx_failure_rate` are
specifically the signal that tells you a show has "gone hot" in real time
— a spike here, correlated with one `showId`, is the operational signature
of the exact scenario `07-concurrency-design.md` is built to survive.

## Health Checks

Each service exposes `/actuator/health` (Spring Boot Actuator) with
liveness (process up) and readiness (dependencies — DB, Redis, Kafka
reachable) checks, used by the load balancer/orchestrator to route traffic
only to instances that can actually serve it.

## What Gets Alerted On

| Signal | Why it matters |
|---|---|
| `outbox_unpublished_backlog` growing unboundedly | Kafka is down or the publisher is stuck — downstream systems are silently falling behind |
| `kafka_consumer_lag` on `payment-events` | Bookings are staying `PENDING_PAYMENT` longer than expected despite payment success |
| `circuit_breaker_state=open` on Booking→Payment | User-facing checkout is actively degraded |
| Sustained `booking_hold_total{result="conflict"}` spike on one `showId` | Expected/healthy signal during a hot on-sale moment — dashboarded, not necessarily alerted, but visible |
