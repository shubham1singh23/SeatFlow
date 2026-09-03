# 07 — Concurrency Design (Core Document)

## The Guarantee

> For a given `(show_id, seat_id)`, at most one booking may ever reach
> `CONFIRMED` state.

This must hold across multiple app instances, retries, duplicate requests,
crashes mid-flow, and partial infrastructure failure.

## Options Considered

| Option | Description | Verdict |
|---|---|---|
| **1. Optimistic locking (version column) only** | Read seat row, write with `WHERE version = ?`, retry on conflict. | Correct in isolation, but under real contention (30 requests for 1 seat) it produces a thundering herd of retries all hitting Postgres. Doesn't give fast, cheap rejection. |
| **2. Pessimistic DB locking (`SELECT ... FOR UPDATE`)** | Lock the row for the duration of the hold decision. | Correct, but holding a row lock across a network round-trip is dangerous, and it doesn't scale well when many *different* seats are contended across many *different* transactions all hammering the same table's lock manager. |
| **3. DB constraint only (unique index), no pre-check** | Just try to insert/update and let the constraint reject duplicates. | Correct as a **backstop**, but on its own gives every losing request an expensive failed transaction instead of a fast rejection, and doesn't handle the *temporary hold* concept (a hold isn't a confirmed row yet). |
| **4. Redis atomic reservation (`SET NX EX`)** | First writer of `hold:{showId}:{seatId}` wins; TTL auto-expires abandoned holds. | Fast, cheap, naturally expresses "temporary." But Redis is not durable/transactional with Postgres — a crash between the Redis `SET` and the Postgres write, or a Redis failover, can create disagreement. Cannot be the *only* guarantee. |
| **5. External distributed lock service (e.g. ZooKeeper/etcd)** | General-purpose distributed mutex. | Correct but heavyweight; introduces a whole new stateful system for a problem Redis + a DB constraint already solve more cheaply. |
| **6. Hybrid: Redis fast-path + Postgres constraint backstop** | Redis decides who *gets to try* to hold; Postgres transaction + partial unique index decides who *actually* gets confirmed. | **Selected.** |

## Decision: Hybrid — Redis Fast Reservation + Postgres as Durable Truth

```
                 User A                         User B
                    │                              │
                    ▼                              ▼
        SET hold:{show}:{seat} NX EX 300s   SET hold:{show}:{seat} NX EX 300s
                    │                              │
             (Redis is single-threaded;             │
              exactly one SET succeeds)              │
                    │                              │
              SUCCESS                            FAILS (key exists)
                    │                              │
      Postgres TX:                          409 SEAT_UNAVAILABLE
      UPDATE show_seat_inventory                    │
      SET status='HELD', hold_id=?, version=version+1
      WHERE show_id=? AND seat_id=? AND status='AVAILABLE'
      AND version = <expected>
                    │
        0 rows updated? (lost race in Postgres too,
        e.g. Redis key expired + someone else already
        confirmed) -> release Redis key, return 409
                    │
        1 row updated -> hold confirmed, return 201
```

### Why two layers, not one

- **Redis alone** is fast but not durable-consistent with Postgres — a
  Redis failover or an expired-but-not-yet-cleaned-up key could let two
  "winners" through in rare edge cases.
- **Postgres alone** (via `UPDATE ... WHERE status='AVAILABLE'`, an
  atomic conditional write) is *fully correct by itself* — it's a
  single-row, single-statement, atomic compare-and-swap. But under heavy
  contention on one popular seat, that funnels dozens of concurrent
  transactions at the same row, all doing real transactional work only to
  have all-but-one fail. Redis's `SET NX` acts as a **cheap admission
  filter** upstream of Postgres: it turns "30 concurrent DB transactions
  fighting over one row" into "1 DB transaction, 29 near-instant in-memory
  rejections."

**The actual correctness guarantee is the Postgres conditional update /
partial unique index — Redis is a performance optimization on top of an
already-correct mechanism, not a replacement for it.** If Redis is
completely unavailable, the system falls back to Postgres-only compare-
and-swap (degraded throughput under contention, but *never* incorrect —
see the Redis-failure row in `11-reliability-and-failure-handling.md`).

## Full Confirmed-Booking Guarantee

The Redis+Postgres dance above governs the **hold**. The final guarantee
against double-booking is enforced independently, a second time, at
**confirmation**:

```sql
CREATE UNIQUE INDEX uq_confirmed_booking_seat
  ON booking_seats (show_id, seat_id)
  WHERE status = 'CONFIRMED';
```

Even if two holds somehow both believed they owned the same seat (a bug,
an edge case, an operator error replaying data), only one `INSERT`/`UPDATE`
transitioning a `booking_seats` row to `CONFIRMED` for that `(show_id,
seat_id)` can ever commit. The second is rejected by Postgres itself with
a unique-violation, at the database engine level — no application logic
has to get this right for the guarantee to hold. This is deliberate
defense in depth: **the hold path is optimized to make double-booking rare
and fast to reject; the confirm path makes it structurally impossible.**

## Step-by-Step: User A vs User B on Seat A12

1. Both requests hit different Booking Service instances (stateless,
   horizontally scaled) at ~the same time.
2. Both attempt `SET hold:show123:seatA12 <holdId> NX EX 300`.
3. Redis is single-threaded per key — exactly one `SET` succeeds.
   Loser gets `nil` back immediately → `409` returned to User B in
   single-digit ms, no DB round-trip.
4. Winner (User A) performs the Postgres conditional `UPDATE`
   (`WHERE status='AVAILABLE'`). If Catalog/Booking state was somehow
   already stale (extremely rare — see failure table), this can still
   fail; in that case A also gets `409` and the Redis key is released.
5. On success, a `hold_id` (UUID) is returned to User A with a 5-minute
   TTL/expiry, mirrored in both Redis (auto-expiry) and Postgres
   (`holds.expires_at`, reconciled by a lightweight sweeper job — see
   below).
6. User A proceeds to payment. On payment success (via webhook →
   Payment Service → event → Booking Service), Booking Service runs the
   confirm transaction: `holds.status='ACTIVE'` check, insert/update
   `booking_seats` to `CONFIRMED`, protected by the partial unique index
   above, in one ACID transaction.
7. If User A abandons checkout, the Redis key TTL expires (300s) and a
   **hold-expiry sweeper** (scheduled job scanning
   `holds WHERE status='ACTIVE' AND expires_at < now()`) transitions the
   Postgres seat row back to `AVAILABLE` and the hold to `EXPIRED` — this
   is the reconciliation path for when Redis expiry and Postgres state
   could otherwise drift.

## Application Crash / Redis Failure / DB Failure During This Flow

| Scenario | Outcome |
|---|---|
| App crashes after Redis `SET`, before Postgres `UPDATE` | Redis key exists but no corresponding Postgres HELD row. TTL expires it in ≤300s; worst case, the seat is unavailable for up to 5 minutes when it should have been bookable — an availability cost, never a correctness violation. |
| App crashes after Postgres `UPDATE`, before response sent to client | Seat is correctly HELD in Postgres. Client retries with the **same** `Idempotency-Key`; the hold endpoint is idempotent (see below) and returns the existing hold rather than creating a second one. |
| Redis fully down | Fast-path admission filter is skipped; falls back to Postgres-only conditional `UPDATE`. Still fully correct, just slower/more contended under a hot-seat spike. |
| Postgres primary down | Booking writes fail outright (by design — we do not allow bookings to be created against a system that cannot durably guarantee the constraint). Reads can still be served from a replica. This is the one place the system chooses **consistency over availability**, deliberately (CAP trade-off, stated explicitly). |

## Idempotency on the Hold/Booking Endpoints

Every `POST /bookings/holds` and `POST /bookings` request carries a
client-generated `Idempotency-Key`. The Booking Service stores
`(idempotency_key) → result` for a bounded window (e.g. 24h). A retried
request with the same key returns the **original** result rather than
re-executing the operation — this is what makes "network timeout, client
retries" safe (see `11-reliability-and-failure-handling.md`).

## Why Not Just Pessimistic Locking Everywhere

`SELECT ... FOR UPDATE` was rejected as the *primary* mechanism because
holding a Postgres row lock across the time it takes to also touch Redis
and build a response is exactly the anti-pattern that turns brief
contention into a queueing collapse under a hot-show spike. The chosen
design keeps the actual lock hold-time to a single, fast, unconditional
statement (`UPDATE ... WHERE status='AVAILABLE'`), executed only once
Redis has already filtered out the losers.
