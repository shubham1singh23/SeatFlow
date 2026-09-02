# 03 — System Architecture

## Monolith vs Modular Monolith vs Microservices

| Criterion | Monolith | Modular Monolith | Microservices |
|---|---|---|---|
| Scalability | Whole app scales together — wasteful when only booking is hot | Same limitation | Each service scales independently (critical: Booking scales differently than Notification) |
| Deployment | Single deploy, simple | Single deploy, simple | Independent deploys, higher ops cost |
| Failure isolation | A bug in Notification can take down Booking | Same process = same blast radius | A Notification outage cannot block Booking |
| Team ownership | Fine at small scale | Good compromise | Best for clear domain ownership |
| Data consistency | Trivial (single DB, local transactions) | Trivial if disciplined | Hard — requires outbox/events across service boundaries |
| Operational overhead | Lowest | Low | Highest (multiple DBs, brokers, deployments) |

### Decision

**Microservices — but a deliberately small set (5 services), not a
service-per-entity decomposition.** The scale numbers in
`02-scale-estimation.md` do not *require* microservices for throughput. The
justification is **failure isolation and independent scaling of the
booking-critical path**: Catalog browsing must stay available even if
Payment or Notification degrade, and the Booking service — the component
carrying the correctness guarantee — must not share fate with lower-stakes
components like Notification.

A **modular monolith** was the strongest alternative and is explicitly
called out in `decisions/ADR-001-microservices.md` as the "if I were
optimizing purely for simplicity" choice — rejected here specifically to
demonstrate independent scaling/failure isolation for a portfolio project,
not because it's the only defensible answer for a real team of this size.

We reject **15+ microservices** (e.g. separate Movie, Venue, Screen, Show,
Pricing services) because none of those sub-domains have independent
scaling, ownership, or failure-isolation needs from each other — they are
one bounded context (Catalog) with one consistency domain.

## Canonical Service List

| Service | Owns | Consistency Domain |
|---|---|---|
| **Identity Service** | Users, credentials, tokens | Strong (its own DB) |
| **Catalog Service** | Movies/Events, Venues, Screens, physical Seats, Shows, Pricing | Strong within catalog writes; heavily cached for reads |
| **Booking Service** | Seat state per show (AVAILABLE/HELD/BOOKED), Holds, Bookings | **Strong — this is the correctness-critical service** |
| **Payment Service** | Payment intents, provider integration, webhook processing, refunds | Strong (its own DB), coordinates with Booking via events |
| **Notification Service** | Delivery of email/SMS/push | Eventually consistent, at-least-once, best-effort |

Booking and Inventory are **one service, not two**. Splitting "check
availability," "hold," and "confirm" across separate services would force a
distributed transaction on the exact critical path we most need to be fast
and correct. Owning seat state end-to-end in one service with one database
is what lets us use a single ACID transaction + one unique constraint as the
ultimate correctness backstop (see `07-concurrency-design.md`).

## High-Level Component Diagram

```
Clients (Web / Mobile)
        │
        ▼
   Load Balancer
        │
        ▼
   API Gateway  ── AuthN/AuthZ, rate limiting, routing
        │
   ┌────┼─────────────┬───────────────┬────────────────┐
   ▼    ▼              ▼               ▼                ▼
Identity  Catalog    Booking         Payment        Notification
  Svc      Svc         Svc             Svc              Svc
   │        │            │               │                │
   │        │      ┌─────┴─────┐         │                │
   │        │      │           │         │                │
   │        │    Postgres    Redis   Postgres          (no DB;
   │        │   (Booking)  (holds)  (Payment)           consumes
   │        │      │           │         │              events)
Postgres  Postgres  │           │         │
(Identity)(Catalog) └─────┬─────┴────┬────┘
              Cache (Redis, read-through)
                          │
                        Kafka  ── domain events, outbox relay
                          │
              ┌───────────┴────────────┐
              ▼                        ▼
      External Payment Provider   External Notification Provider
```

See `diagrams/02-high-level-architecture.drawio` for the visual version.

## Communication Patterns

| Path | Style | Why |
|---|---|---|
| Client → API Gateway → Services | Synchronous REST | User-facing request/response |
| Booking Service → Payment Service | Synchronous REST (create payment intent) | User is waiting for a redirect/next step |
| Payment Provider → Payment Service | Async webhook (HTTP callback) | Provider-controlled timing, must be idempotent |
| Booking/Payment → Kafka (Outbox) | Async, at-least-once | Decouple confirmation side-effects from the critical path |
| Kafka → Notification Service | Async consumer | Notifications must never block booking/payment |
| Catalog reads | Sync REST, cache-first | High read volume, tolerant of short staleness |

Synchronous calls are used only where the caller genuinely cannot proceed
without an answer (payment intent creation). Everything that is a
*side-effect* of a state change (notify user, update read models) is
asynchronous via Kafka.
