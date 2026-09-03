# ADR-005: Transactional Outbox for Event Publication

## Context
Booking state changes must reliably reach Notification and Analytics
Services via Kafka, without losing events on crash and without a
distributed transaction spanning Postgres and Kafka.

## Problem
A naive "commit DB, then publish to Kafka" has a window where the DB
commits but the publish fails/never happens — silent inconsistency.

## Options considered
- Direct produce-after-commit (no outbox) — rejected, exactly the
  dual-write problem above.
- Two-phase commit / XA across Postgres and Kafka — rejected, Kafka has no
  mature, low-risk XA support, and 2PC's operational cost is high for the
  guarantee it buys.
- Transactional Outbox table + separate relay process — selected.

## Decision
Transactional Outbox, as detailed in
`docs/lld/11-event-and-outbox-design.md`.

## Why we chose it
It reuses infrastructure we already have (Postgres transactional
guarantees) instead of requiring Kafka to participate in a distributed
transaction it isn't well-suited for, and it produces a well-understood,
at-least-once delivery contract that consumers can build against.

## Trade-offs
At-least-once, not exactly-once — consumers must dedupe on `event_id`.
Adds a relay process (operational component) and replication lag between
"booking confirmed" and "event visible in Kafka" (bounded by the relay's
poll interval).

## Consequences
Every write path that needs to emit an event must go through
`OutboxService.record(...)` inside its existing transaction — enforced via
`Propagation.MANDATORY` so a future contributor can't accidentally call it
outside a transaction and silently break the atomicity guarantee.

## Failure scenarios
Relay crash mid-batch, Kafka unavailability — both covered in
`docs/lld/12-error-and-failure-handling.md`; both result in eventual,
possibly-duplicate delivery, never lost delivery of a committed change.

## Future evolution
If outbox-table growth or relay latency becomes a bottleneck, a
CDC-based relay (e.g. Debezium tailing the WAL) is the natural next step —
same guarantee, different relay implementation, no schema change required.
