# ADR-002: PostgreSQL as Primary Database for All Services

## Context
Every service needs durable storage. Booking Service specifically needs a
mechanism to structurally guarantee "at most one CONFIRMED booking per
seat per show."

## Problem
Which persistence technology gives the strongest, cheapest correctness
guarantee for the booking write path, while remaining a sound general
choice for the other services?

## Options Considered

### Option 1: PostgreSQL
Full ACID, partial/expression unique indexes, mature replication.

### Option 2: MySQL
Full ACID, unique indexes, but no partial (filtered) unique indexes —
would require a workaround (e.g. a separate always-populated column) to
express "unique only among CONFIRMED rows."

### Option 3: MongoDB
Flexible schema, strong single-document consistency, weaker native support
for the exact relational constraint needed; would push "one confirmed
booking per seat" enforcement into application logic instead of the
database engine.

## Decision
PostgreSQL, one database per service.

## Why We Chose It
The core guarantee of this entire project is enforceable as a single
partial unique index (`WHERE status = 'CONFIRMED'`) — a database-engine-
level guarantee that doesn't depend on application code being correct
under concurrency. This is the strongest, cheapest version of that
guarantee available among the options. Postgres's `SELECT ... FOR UPDATE
SKIP LOCKED` and rich isolation-level controls were also considered useful
for potential future admin/reporting concurrency needs.

## Trade-offs
- Less schema flexibility than MongoDB for rapidly evolving catalog
  attributes (mitigated: JSONB columns for genuinely dynamic fields like
  movie metadata).
- One database technology to operate well, rather than picking "best tool
  per service" — accepted in exchange for operational simplicity and
  consistent team expertise.

## Consequences
- `bookings`/`payments` tables need explicit partitioning strategy given
  projected growth (see `05-data-architecture.md`).
- Every service gets its own Postgres instance/schema — no cross-service
  joins are possible by construction, which reinforces the "no shared DB
  access" service-boundary rule (ADR-001).

## Failure Scenarios
- Booking DB down → booking writes hard-fail rather than risk an
  unenforceable guarantee (see `11-reliability-and-failure-handling.md`).

## Future Evolution
If a single write-heavy `show_seat_inventory`/`bookings` table ever
becomes a real bottleneck (not just a theoretical one — see
`13-scalability-strategy.md`), horizontal sharding by `show_id` range is
the natural next step within Postgres, not a switch to another database.
