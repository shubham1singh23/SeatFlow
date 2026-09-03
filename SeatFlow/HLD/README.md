# SeatFlow

**Distributed Ticket Booking Platform — HLD / Architecture Repository**

## Problem

SeatFlow lets users discover movies/events and book seats; venue managers
manage the catalog. The central engineering problem is **concurrent seat
allocation**: when many users race for the same seat on the same show,
exactly one may win — under crashes, retries, duplicate requests, and
partial failure of Redis, Kafka, or Postgres. See
[`docs/07-concurrency-design.md`](docs/07-concurrency-design.md).

## Engineering Challenges Addressed

- Preventing double-booking under real concurrency, including hot-show
  spikes (dozens of simultaneous requests for one seat)
- Keeping payment state and booking state consistent across service and
  network boundaries (the dual-write problem)
- Making every retry, duplicate request, and duplicate event safe
  (idempotency end to end)
- Isolating failure domains so a low-stakes outage (Notification) can
  never affect a high-stakes path (Booking/Payment)
- Scaling read-heavy browsing independently from write-sensitive booking

## Requirements & Scale Assumptions

- [`docs/01-requirements.md`](docs/01-requirements.md)
- [`docs/02-scale-estimation.md`](docs/02-scale-estimation.md) *(explicit design assumptions, not measured stats)*

## Architecture

5 services, each owning its own PostgreSQL database, communicating via
REST (sync) and Kafka (async side-effects):

| Service | Owns | Consistency |
|---|---|---|
| Identity | users, credentials, tokens | Strong |
| Catalog | movies, venues, screens, seats, shows, pricing | Cached, eventually-fresh reads |
| **Booking** | seat state per show, holds, bookings | **Strong — correctness-critical** |
| Payment | payment intents, payments, refunds | Strong |
| Notification | delivery log | Eventually consistent, isolated blast radius |

- [`docs/03-system-architecture.md`](docs/03-system-architecture.md)
- [`docs/04-service-boundaries.md`](docs/04-service-boundaries.md)
- [`docs/05-data-architecture.md`](docs/05-data-architecture.md)
- [`docs/06-api-architecture.md`](docs/06-api-architecture.md)

## Concurrency Model — How Double-Booking Is Prevented

**Hybrid: Redis `SET NX EX` as a fast admission filter, Postgres
conditional `UPDATE` as the real hold guarantee, and a Postgres partial
unique index as a structural, database-engine-level backstop at
confirmation time.** Redis is never the durable source of truth — it's an
optimization that makes the already-correct Postgres mechanism fast under
hot-seat contention.

Full reasoning, failure walkthroughs, and the "User A vs User B" trace:
[`docs/07-concurrency-design.md`](docs/07-concurrency-design.md) ·
[`decisions/ADR-004-concurrency-control.md`](decisions/ADR-004-concurrency-control.md) ·
diagram: [`diagrams/05-seat-concurrency.drawio`](diagrams/05-seat-concurrency.drawio)

## Data Consistency

Strong consistency for seat state, bookings, payments. Eventual
consistency for notifications and catalog browsing cache. This boundary is
stated explicitly and enforced by named mechanisms throughout — see
[`decisions/ADR-010-consistency-model.md`](decisions/ADR-010-consistency-model.md).

## Event-Driven Architecture

Kafka carries cross-service side effects only (never the hold decision
itself), via the **Transactional Outbox Pattern** to avoid the dual-write
problem, with idempotent consumers and per-topic DLQs.

- [`docs/09-event-driven-architecture.md`](docs/09-event-driven-architecture.md)
- [`decisions/ADR-006-kafka-event-driven-processing.md`](decisions/ADR-006-kafka-event-driven-processing.md)
- [`decisions/ADR-007-outbox-pattern.md`](decisions/ADR-007-outbox-pattern.md)

## Payment Architecture

Idempotent payment intents, HMAC-verified/idempotent webhook handling,
reconciliation against the provider for stuck intents, and outbox-driven
booking confirmation on payment success.

- [`docs/10-payment-architecture.md`](docs/10-payment-architecture.md)

## Reliability

A full failure matrix (Redis down, Kafka down, DB down, service crash,
duplicate requests/webhooks/events, payment-succeeds-booking-fails, and
more), each resolved by idempotency, at-least-once + idempotent consumers,
or a deliberate, stated availability trade-off.

- [`docs/11-reliability-and-failure-handling.md`](docs/11-reliability-and-failure-handling.md)

## Scalability

Stateless, horizontally-scaled services; an explicit analysis of why
adding instances does *not* solve hot-seat contention (that's a data-layer
problem, solved separately); 1x/10x/100x traffic walkthrough.

- [`docs/13-scalability-strategy.md`](docs/13-scalability-strategy.md)

## Security

JWT auth + RBAC, bcrypt password hashing, two-layer input validation,
Redis-backed rate limiting, HMAC-verified webhooks — with an explicit
"implemented vs. production-hardening recommendation" split (no
overclaiming).

- [`docs/12-security-architecture.md`](docs/12-security-architecture.md)

## Observability

Correlation IDs, OpenTelemetry tracing, Prometheus/Grafana metrics
including business metrics like `booking_hold_total{result="conflict"}` —
the direct operational signal for a show "going hot."

- [`docs/14-observability.md`](docs/14-observability.md)

## Architecture Diagrams

10 real, editable Draw.io diagrams in [`diagrams/`](diagrams/), each with
a direct `app.diagrams.net` link — see
[`diagrams/README.md`](diagrams/README.md). No installs, no MCP server,
no external generation service: each link is built directly from the
`.drawio` XML using the standard, documented `#R<xml>` diagrams.net URL
scheme.

| # | Diagram | Answers |
|---|---|---|
| 01 | System Context | What does SeatFlow look like from the outside? |
| 02 | High-Level Architecture | The primary "explain the whole system in 60s" diagram |
| 03 | Service Architecture | Who owns what, who calls whom |
| 04 | Booking Sequence | The full hold → pay → confirm → notify flow |
| 05 | Seat Concurrency | **Why double-booking is structurally impossible** |
| 06 | Payment Flow | Idempotency and duplicate-webhook handling |
| 07 | Event Flow | Kafka topics, consumer groups, DLQs |
| 08 | Database Architecture | Per-service data ownership (no shared DBs) |
| 09 | Failure Handling | Recovery mechanism per failure type |
| 10 | Deployment Architecture | LB → gateway → services → data stores, horizontal scaling |

## Architectural Decisions

10 ADRs in [`decisions/`](decisions/), each with options considered,
rejected alternatives, explicit trade-offs, and failure scenarios — not
just "we picked X because it's popular":

`ADR-001` microservices · `ADR-002` database · `ADR-003` seat inventory
model · `ADR-004` concurrency control · `ADR-005` Redis seat holds ·
`ADR-006` Kafka · `ADR-007` outbox pattern · `ADR-008` API communication ·
`ADR-009` caching · `ADR-010` consistency model

## HLD Trade-offs

A consolidated "what was gained, what was given up" table across every
major decision, plus how the design would change at a different scale:
[`docs/15-hld-tradeoffs.md`](docs/15-hld-tradeoffs.md).

## Performance Testing

**Not yet implemented.** The design targets in `01-requirements.md` are
stated explicitly as design assumptions, not measured results. Load
testing (e.g. simulating the blockbuster hot-seat scenario from
`02-scale-estimation.md` against the Redis+Postgres hold mechanism) is the
natural next step to validate this design empirically — noted as *future
implementation*, not fabricated here.

## Future Evolution

- Sharding `show_seat_inventory`/`bookings` by `show_id` range if a
  single Postgres primary's write throughput ever becomes a real (not
  theoretical) bottleneck.
- CDC-based outbox relay (e.g. Debezium) if outbox polling latency
  becomes measurable.
- A pre-hold virtual waiting-room/admission-control layer for extreme
  blockbuster on-sale moments, additive to (not a replacement for) the
  current concurrency mechanism.
- mTLS between internal services, WAF/DDoS at the edge, secrets rotation
  automation — flagged as production-hardening gaps in
  [`docs/12-security-architecture.md`](docs/12-security-architecture.md).

---

## Repository Structure

```
SeatFlow/
├── README.md                    -- this file
├── docs/                        -- 15 architecture documents (01-15)
├── diagrams/                    -- 10 editable .drawio diagrams + link index
└── decisions/                   -- 10 ADRs
```
