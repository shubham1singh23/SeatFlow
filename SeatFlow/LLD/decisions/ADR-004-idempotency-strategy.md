# ADR-004: Idempotency Strategy

## Context
Mobile/flaky-network clients will retry booking commands; payment
callbacks are inherently at-least-once. Both need idempotency.

## Problem
How do we guarantee "same request repeated N times → executes once,
returns the same result N times" — including when two identical requests
race in truly concurrently, not just sequentially?

## Options considered
- Application-level in-memory dedup cache (per instance) — rejected,
  doesn't survive restarts or work across instances.
- Redis-only dedup key — rejected as sole mechanism for the same reason
  Redis is not the sole seat-claim authority (ADR-001): not durable enough
  to be the only guarantee for a "did we already charge/book this"
  question.
- Postgres `idempotency_records` table with a composite primary key as
  the concurrency-resolution mechanism itself.

## Decision
Postgres-backed `idempotency_records`, PK = `(idempotency_key, user_id,
operation)`. See `docs/lld/09-idempotency.md`.

## Why we chose it
Same reasoning pattern as ADR-001: let the database's own uniqueness
guarantee resolve the race, rather than building a custom
lock/coordination mechanism in application code.

## Trade-offs
An extra DB round-trip on every mutating request; judged worth it given
the alternative is either incorrect under concurrency or requires its own
distributed-lock reasoning (which we already decided in ADR-001 to avoid
where a constraint will do the job).

## Consequences
Every mutating endpoint requires the header; documented explicitly in
`docs/lld/06-api-design.md` per endpoint.

## Failure scenarios
Covered in `docs/lld/09-idempotency.md` (concurrent duplicates, key reuse
with different payload, TTL expiry).

## Future evolution
None needed at current scope.
