# ADR-007: Transactional Outbox Pattern

## Context
Booking and Payment services both need to atomically (a) update their own
database and (b) reliably notify the rest of the system via Kafka. These
are two different systems with no shared transaction.

## Problem
The dual-write problem: if the DB commit succeeds and the Kafka publish
fails independently, the rest of the system silently never learns about a
state change that actually happened.

## Options Considered

### Option 1: Direct dual-write (update DB, then publish to Kafka)
Simple, but exactly the failure mode above — a Kafka publish failure after
a successful DB commit is silent data-loss from the rest of the system's
perspective.

### Option 2: Transactional Outbox
Write the business row and an outbox row in the same local ACID
transaction; a separate publisher process reads unpublished outbox rows
and publishes them, retrying until acknowledged.

### Option 3: Change Data Capture (CDC) on the DB transaction log
Similar guarantee to Option 2, sourced from the DB's write-ahead log
instead of an explicit outbox table (e.g. Debezium). More infrastructure
to operate; considered but not required at this project's scope.

## Decision
Option 2 — an explicit `*_outbox` table per service, with a polling
publisher.

## Why We Chose It
Gives the exact guarantee needed (event exists if and only if the
business state change committed) using only infrastructure already in the
stack (Postgres, Kafka), without adding a CDC pipeline's operational
weight for a project at this scale. CDC (Option 3) is noted as the
natural evolution if outbox-table polling ever becomes a measurable
bottleneck.

## Trade-offs
- Adds a small latency window between "state committed" and "event
  published" (bounded by publisher poll interval).
- One more table + one more background process per service that owns
  events.

## Consequences
- Downstream consumers must be idempotent (ADR-006) since the outbox
  guarantees at-least-once, not exactly-once, delivery.
- Publisher failures/backlogs are directly observable via
  `outbox_unpublished_backlog` metric (`14-observability.md`).

## Failure Scenarios
- Kafka down: outbox rows accumulate `published=false`; no data loss,
  only delayed downstream effects. Fully covered in
  `11-reliability-and-failure-handling.md`.

## Future Evolution
Migrate to CDC-based outbox relay (e.g. Debezium) if polling latency or
publisher load ever becomes a genuine bottleneck — noted as future
implementation, not needed at current scale.
