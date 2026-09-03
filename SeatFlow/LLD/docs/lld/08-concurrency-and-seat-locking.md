# 08 — Concurrency and Seat Locking

## The invariant we must protect

> For a given `(showId, seatId)`, at most one `BookingItem` may be in an
> **active** status (`HELD` or `CONFIRMED`) at any time — across any number
> of Booking Service instances.

## Why "just lock the seat" doesn't type-check for this service

A pessimistic lock (`SELECT ... FOR UPDATE`) needs a row to lock. Booking
Service does **not** own a `seats` table — seat master data lives in Show
Service (see `01-scope-and-responsibilities.md`). Locking a row Booking
Service doesn't own, in a database it doesn't own, is not on the table.
So the mechanism has to be expressed in terms of a row Booking Service
*does* own: `booking_items`.

## Options considered

| Option | Correctness | Multi-instance safe | Failure behavior | Perf | Complexity | Ops cost | Recovery | Debuggability | Scalability |
|---|---|---|---|---|---|---|---|---|---|
| A. App-level (in-JVM) lock | ❌ only within one instance | ❌ | silent double-book across instances | good | low | low | n/a | poor (false confidence) | ❌ |
| B. Redis distributed lock (Redlock-style, lock around a critical section) | ⚠️ correctness depends on clock/GC assumptions Redlock is known to be fragile under | ✅ if implemented correctly | lock can be lost mid-critical-section (GC pause, network partition) → **double booking possible** | good | high (lock renewal, fencing tokens) | medium | manual | medium | good |
| C. Redis `SET NX PX` as a hold *claim*, no DB constraint | ✅ while Redis is up | ✅ | **Redis is a single point of correctness** — data loss/failover can silently double-allocate | excellent | low | low (until you need HA correctness, then high) | poor (nothing durable) | poor | good |
| D. PostgreSQL pessimistic lock (`SELECT ... FOR UPDATE`) on a seat-claim row | ✅ | ✅ | lock held for txn duration; contention serializes hot seats | poor under hot contention (popular show = lock queue) | medium | low | good (rolls back) | good | poor at extreme contention |
| E. PostgreSQL optimistic locking (version column, retry on conflict) | ✅ | ✅ | conflict = retry loop; wasted work under very high contention | good at moderate contention, degrades under extreme hot-seat contention | medium | low | good | good | medium |
| F. DB conditional insert / **unique constraint** | ✅ (DB is the invariant's enforcer, by definition) | ✅ | conflict = clean, fast constraint-violation error, no lock held | excellent (no lock, no polling) | low | low | good (nothing to roll back but the failed insert) | excellent (constraint violation is unambiguous) | excellent |
| G. **Hybrid: Redis fast-path admission + Postgres unique constraint as authority** | ✅ (F's guarantee, never weakened) | ✅ | if Redis is down, falls back to F alone — still correct, just slower | excellent (Redis avoids DB round-trip for seats obviously taken) | medium | medium | good | good | excellent |

## Decision: **Option G**

Rationale, directly against the invariant:

1. **Option F alone is already correct and sufficient.** A partial unique
   index `UNIQUE (show_id, seat_id) WHERE status IN ('HELD','CONFIRMED')`
   on `booking_items` makes the invariant a database-enforced fact, not a
   Java-enforced convention. `CreateBooking` simply attempts to `INSERT`
   a `BookingItem` per requested seat inside one transaction; if any
   insert violates the constraint, the whole transaction rolls back (see
   §"all-or-nothing" below) and the caller gets a clean 409. No lock is
   *held* — the constraint is checked at insert time, so there's nothing
   to leak on crash, GC pause, or network partition. This directly beats
   Option B/C, which each introduce a coordination mechanism that can be
   wrong *while looking healthy*.

2. **Why add Redis at all, then (Option G over plain F)?** Under a
   blockbuster on-sale event, thousands of users may hit "obviously
   already taken" seats in the same second (everyone refreshing the seat
   map and clicking the same few "good" seats). Sending every one of
   those attempts to Postgres as a doomed `INSERT` wastes DB connections
   and increases p99 latency for the users who *do* have a real shot.
   `SeatHoldCoordinator.tryClaim(showId, seatId, holdToken)` issues a
   Redis `SET key NX PX <ttlMillis>` per seat as a cheap **admission
   filter**: if Redis says "already claimed," we reject in ~1ms without
   touching Postgres. If Redis says "claimed by you," we proceed to the
   authoritative DB insert. Redis never gets to say "yes" on its own
   without the DB confirming — so Redis being wrong (stale, evicted, down)
   can only cause an unnecessary DB round-trip, never a double booking.

3. **Redlock-style locking (B) is explicitly rejected**, not because Redis
   is bad, but because a *lock held across a critical section* has a
   correctness dependency on timing (lease survives the whole critical
   section) that this design refuses to depend on. Using Redis as a
   single atomic `SET NX` **claim check**, with Postgres as the final
   arbiter, sidesteps that whole class of problems — we get Redis's speed
   without inheriting Redlock's fragility.

4. **Pessimistic DB locking (D)** was rejected because it holds a lock for
   the transaction's lifetime, which under hot-seat contention becomes a
   queue — and we don't need a queue, we need a fast **reject**. A unique
   constraint gives the same correctness with a rejection instead of a
   wait.

5. **Optimistic locking (E)** is close to F but is designed for
   *update* conflicts on a row that already exists (e.g. re-confirming a
   payment against the `bookings.version` column, which we **do** use —
   see `07-database-design.md`). For the *first claim* of a seat there is
   no existing row to version against, so a unique constraint on insert is
   the more direct tool.

## "If Redis disappears, can the system still preserve booking correctness?"

**Yes, unconditionally.** Redis in this design is a *coordination cache*,
never a source of truth:

```
Redis   = temporary / admission-control state (may be lost, stale, or absent)
Postgres = durable source of truth for booking + seat-claim state
```

If Redis is unreachable, `SeatHoldCoordinator.tryClaim` fails open to
"proceed to DB" (never fails open to "seat is free" — see
`12-error-and-failure-handling.md`), so every attempt falls through to the
unique-constraint check in Postgres. Throughput drops (more DB round-trips
on hot seats), but the invariant — the thing that actually matters — is
never at risk.

## All-or-nothing multi-seat acquisition

For a request of `[A1, A2, A3, A4]` where `A3` is already taken: the
**entire booking fails**, not a partial 3-of-4 hold.

Why: ticket buyers book seats *together* (a group wants to sit together);
silently returning 3 seats when 4 were asked for is a worse UX than a
clear "not all seats available, please reselect" and creates orphaned
holds a user never wanted. Mechanically this falls out for free from
Option G: all `BookingItem` inserts happen in **one DB transaction** — a
single constraint violation on any seat rolls back the whole insert set,
so there is no window where A1/A2 are held while A3 fails. The Redis
admission-check loop (step 2 above) is also short-circuited on first
rejection, so we don't even attempt DB writes for the remaining seats once
one has failed.

## Multi-instance behavior

Nothing above is instance-local. Redis and Postgres are both shared
infrastructure; any of N Booking Service instances calls the same
`SeatHoldCoordinator` (same Redis) and the same Postgres unique index.
Going from 1 → 20 instances changes nothing about correctness — it only
increases the *rate* of concurrent attempts the mechanism must reject
correctly, which is exactly what Option G is built to do without a shared
in-process lock.
