# 15 — HLD Trade-offs Summary

A consolidated view of the major decisions and what was deliberately given
up in exchange. Cross-reference `decisions/` for full ADRs.

| Decision | Gained | Given Up |
|---|---|---|
| Microservices (5 services) over modular monolith | Independent scaling & failure isolation of Booking vs. Notification vs. Catalog | Operational complexity: multiple DBs, deployments, network calls where a function call would do in a monolith |
| Booking + Inventory as one service | Single ACID transaction + one unique index gives an airtight correctness guarantee with no distributed transaction | Booking Service is a slightly larger, more critical single point than a maximally-decomposed design |
| PostgreSQL for all services (no MongoDB) | Partial unique indexes as a structural correctness backstop; strong relational integrity | Less schema flexibility for rapidly-changing catalog attributes (mitigated with JSONB columns where genuinely needed) |
| Redis fast-path + Postgres backstop (not Redis-only, not Postgres-only) | Fast rejection under hot-seat contention, with correctness never resting on Redis's durability | More moving parts than a single mechanism; requires care to keep Redis "advisory" and never load-bearing for correctness |
| Transactional Outbox (not direct dual-write) | No silent state divergence between DB and downstream events, even across Kafka outages | Slight latency/complexity added by the outbox-publisher hop; consumers must be idempotent |
| Kafka only for side-effects, not the hold decision | Booking-critical path stays low-latency and synchronous where the user is actually waiting | Kafka's value is narrower in this design than in systems that route everything through events — a deliberate choice, not an oversight |
| Strong consistency for booking/payment; eventual for notifications/search | Correctness where money/inventory is involved, availability where it's merely UX | Notifications can lag; a cache-served seatmap can be a few seconds stale (never trusted for the actual write decision) |
| Consistency over availability for Booking DB outage | Never accept a booking the system can't durably guarantee | Booking writes hard-fail during a Booking-DB outage rather than degrading to a "maybe" state |
| No mTLS / no service mesh in v1 | Simpler v1 build, appropriate for stated threat model | Explicitly flagged as a production-hardening gap, not hidden |

## Where This Design Would Change At A Different Scale

- **10–100x this scale, sustained (not just spiky):** Booking Postgres
  primary write throughput would become the next thing to address —
  natural next step is sharding `show_seat_inventory`/`bookings` by
  `show_id` range, called out as future evolution in `13`.
- **Multi-region, active-active:** would require rethinking the "Postgres
  is single source of truth" model into a per-region-owns-its-shows model
  or a consensus-based seat allocator — explicitly out of scope for this
  design and stated as such rather than hand-waved.
- **Much lower scale (e.g. a single-city, single-cinema-chain MVP):** the
  modular monolith alternative from `03-system-architecture.md` becomes
  the more honestly correct choice; this design's service split is
  justified for a platform with SeatFlow's stated multi-venue, multi-city
  scope, not as a universal default.
