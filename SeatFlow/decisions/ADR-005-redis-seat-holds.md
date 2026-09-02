# ADR-005: Redis TTL-Based Reservation for Temporary Seat Holds

## Context
A seat hold must automatically expire if the user abandons checkout,
without requiring a background process to be the *only* mechanism keeping
inventory correct in real time.

## Problem
How do we represent "this seat is provisionally reserved for the next N
minutes" cheaply and self-cleaning?

## Options Considered

### Option 1: PostgreSQL-only reservation table, cron-based expiry
Correct, but expiry is only as fresh as the sweeper's polling interval,
and doesn't provide the fast in-memory admission-filter benefit described
in ADR-004.

### Option 2: Application memory (in-process timers)
Fails immediately in a horizontally-scaled, stateless fleet — a hold
"owned" by one instance's memory is invisible to every other instance and
lost entirely on instance restart.

### Option 3: Redis with `SET key value NX EX ttl`
Atomic, self-expiring, visible to every instance immediately, and doubles
as the fast admission filter from ADR-004.

### Option 4: Distributed locking service (ZooKeeper/etcd) with TTL leases
Viable but heavyweight for what is fundamentally a simple, high-frequency,
short-TTL key — not proportionate to the problem.

## Decision
Option 3 — Redis `SET NX EX`, key `hold:{showId}:{seatId}`, TTL matching
the hold window (e.g. 300s).

## Why We Chose It
`SET NX EX` is a single atomic operation (no separate "check" then "set"
race window), natively expresses TTL-based expiry without any extra
process, and is visible instantly to every stateless service instance —
exactly the properties needed for a fleet-wide, self-cleaning temporary
reservation.

## Trade-offs
- Redis is not the durable source of truth (see ADR-004) — a Redis
  key expiring does not, by itself, update Postgres; a reconciliation
  sweeper (scanning `holds WHERE status='ACTIVE' AND expires_at < now()`)
  is still needed to keep Postgres's view converged, so Redis TTL alone
  does not eliminate the need for a background job — it just removes that
  job from the *hot path*.

## Consequences
- Hold TTL in Redis and `holds.expires_at` in Postgres must be set
  consistently at hold-creation time (same value, same request).
- The sweeper job is a low-urgency background reconciliation process, not
  a correctness-critical one — worst case if it lags, a released Redis key
  simply means Postgres briefly still shows HELD past actual expiry,
  which is a conservative (safe) direction to be wrong in.

## Failure Scenarios
- Redis restarts/loses the key early (e.g. eviction under memory
  pressure): the seat becomes bookable again "early" — an availability
  cost (seat looks free before Postgres's TTL would have released it,
  but the Postgres row still requires its own conditional check, so this
  never causes a double-booking, only a possible false-negative on hold
  ownership that the sweeper reconciles).

## Future Evolution
None anticipated; this is a narrowly-scoped, well-understood pattern.
