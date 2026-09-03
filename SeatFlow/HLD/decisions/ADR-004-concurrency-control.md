# ADR-004: Hybrid Redis + Postgres Concurrency Control for Seat Holds

## Context
This is the central engineering problem of SeatFlow (see
`07-concurrency-design.md`): guarantee at most one successful hold/booking
per seat per show under real concurrency, including hot-show spikes of
dozens of simultaneous requests for one seat.

## Problem
Which mechanism gives correctness under all failure modes *and* acceptable
performance under extreme single-key contention?

## Options Considered

### Option 1: Optimistic locking (version column) only
Correct, but under heavy contention produces a retry storm hitting
Postgres repeatedly for the same row.

### Option 2: Pessimistic DB locking (`SELECT ... FOR UPDATE`)
Correct, but holding a row lock across additional work (building a
response, further calls) risks queueing collapse under contention; also
doesn't natively express "temporary hold with TTL."

### Option 3: DB unique constraint only, no pre-check
Correct as a backstop, but gives every losing request a full failed
transaction rather than a cheap rejection; doesn't model temporary holds.

### Option 4: Redis atomic reservation only
Fast, TTL-friendly, but not durably consistent with Postgres on its own —
a Redis failover could theoretically let two "winners" through.

### Option 5: External distributed lock (ZooKeeper/etcd)
General and correct, but a new stateful system to operate for a problem
already solvable with infrastructure already in the stack.

### Option 6 (chosen): Redis `SET NX EX` as admission filter + Postgres
conditional `UPDATE` as the real lock + a partial unique index at
confirmation as final backstop.

## Decision
Option 6 — three layered mechanisms, each with a distinct job:
1. Redis: fast rejection of the losing majority under hot-key contention.
2. Postgres conditional `UPDATE ... WHERE status='AVAILABLE'`: the actual
   correctness guarantee for the *hold*.
3. Postgres partial unique index on CONFIRMED bookings: the actual
   correctness guarantee for the *booking*, independent of whether the
   hold logic was perfect.

## Why We Chose It
No single layer alone gives both speed-under-contention and durability-
independent-of-Redis. Layering them lets each do the job it's actually
good at, with the *cheapest* layer (Redis, in-memory) absorbing the bulk
of rejected requests, and the *most durable* layer (Postgres constraint)
being the only thing the correctness guarantee ultimately depends on.

## Trade-offs
- More moving parts than a single mechanism.
- Requires discipline to keep Redis strictly "advisory" — every code path
  must treat a Redis-only success as provisional until Postgres confirms
  it, never as the final answer.

## Consequences
- Redis being down degrades throughput under contention but never
  correctness (fallback to Postgres-only conditional update).
- The confirmation-time unique index makes the system correct even if a
  future bug is introduced in the hold logic — deliberate defense in
  depth.

## Failure Scenarios
See the full walkthrough in `07-concurrency-design.md` (crash between
Redis SET and Postgres UPDATE, Redis down, Postgres down).

## Future Evolution
If a single hot seat/show ever needed *more* than this (e.g. a virtual
waiting-room/queue ahead of the hold endpoint for extreme blockbuster
on-sale moments), that would be an additive admission-control layer in
front of this mechanism, not a replacement for it.
