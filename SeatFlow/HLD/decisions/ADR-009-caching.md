# ADR-009: Redis Read-Through Cache for Catalog, Never for Write Decisions

## Context
Catalog browsing (movies, shows, seatmaps) is read-heavy and must stay
highly available even during a booking-path incident. Booking's write path
must never trust cached data for its correctness decision.

## Problem
Where should caching be applied, and how do we prevent it from ever
becoming a hidden source of truth that undermines the concurrency
guarantee?

## Options Considered

### Option 1: No caching, always hit Postgres
Simplest, but wastes read capacity on largely-static catalog data and
couples browsing availability directly to Catalog DB health.

### Option 2: Read-through cache for Catalog only, short TTL
Absorbs the bulk of browsing reads; Catalog DB only serves cache misses.

### Option 3: Cache used as part of the booking write decision (e.g. trust cached seat status to short-circuit a hold request)
Rejected outright — this would make Redis a second, weaker source of
truth for exactly the state that ADR-004 goes to great lengths to
guarantee correctly. Any caching on the write path must be, at most, an
optimization *upstream of* the real check (which is what the Redis
admission-filter in ADR-004 already is) — never a substitute for it.

## Decision
Option 2 — read-through Redis cache scoped strictly to Catalog's
browsing endpoints (`movies`, `shows`, `seatmap`), short TTL (5–15s) for
inventory-adjacent data, longer TTL for static data. The booking write
path (`07-concurrency-design.md`) has its own, separate use of Redis
(ADR-004/005) that is explicitly not "the Catalog cache."

## Why We Chose It
Keeps the two uses of Redis (browsing cache vs. hold admission filter)
conceptually and operationally distinct, so a caching bug or stale-TTL
issue in browsing can never leak into the correctness-critical booking
decision.

## Trade-offs
- Seatmap displayed to a browsing user can be a few seconds stale — an
  accepted UX trade-off, since the actual hold attempt always re-checks
  live state regardless of what the seatmap displayed.

## Consequences
- No cache-invalidation coupling is required between Booking and Catalog
  services (which would violate the no-shared-DB-access rule) — short TTL
  is the chosen invalidation strategy, deliberately simple.

## Failure Scenarios
- Redis (cache role) down: falls through to Postgres read replica;
  browsing slower, not incorrect, and entirely independent of the Redis
  (hold-admission role) failure behavior in ADR-004.

## Future Evolution
A CDN layer in front of the most static catalog data (movie posters,
descriptions) is a natural additive optimization, not a replacement for
this cache.
