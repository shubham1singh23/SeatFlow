# ADR-001: Seat Concurrency Strategy

## Context
Booking Service must guarantee at most one active claim per (show, seat)
across N horizontally-scaled instances, under bursty, high-contention
traffic (blockbuster on-sale events).

## Problem
Naive approaches either don't work across instances (in-JVM locks) or
introduce a correctness dependency on distributed-lock timing (Redlock)
or on a non-durable system being the sole arbiter (Redis-only claims).

## Options considered
A. App-level lock — B. Redis distributed lock (Redlock-style) —
C. Redis-only claim (no DB backing) — D. Postgres pessimistic lock —
E. Postgres optimistic lock — F. Postgres unique constraint (conditional
insert) — G. Hybrid: Redis admission filter + Postgres unique constraint
as authority.

Full comparison table: `docs/lld/08-concurrency-and-seat-locking.md`.

## Decision
**Option G.** Postgres partial unique index
(`show_id, seat_id WHERE status IN ('HELD','CONFIRMED')`) is the sole
source of correctness. Redis `SET NX PX` is used only as an optional,
fail-open admission filter to reduce DB load from doomed attempts on
obviously-taken seats.

## Why we chose it
It is the only option where the correctness guarantee does not depend on
distributed-lock timing assumptions, and it degrades gracefully (slower,
never wrong) if Redis is unavailable.

## Trade-offs
Two systems in the hot path instead of one; Redis's benefit is purely
latency/DB-load, not correctness, and must be understood that way by
anyone touching this code later.

## Consequences
`SeatHoldCoordinator` must never be allowed to return "available" as a
final answer — only "not fast-path-claimed, proceed to DB" — this
invariant is the one thing a future change to this class must never
violate.

## Failure scenarios
Covered exhaustively in `docs/lld/12-error-and-failure-handling.md`.

## Future evolution
If seat-claim contention ever needs to be resolved *before* it reaches
Postgres at all (e.g. true million-QPS on-sale spikes), a purpose-built
in-memory sequencer per hot show (not Redlock) could be considered — not
needed at SeatFlow's current scale.
