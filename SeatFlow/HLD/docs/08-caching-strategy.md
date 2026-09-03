# 08 — Caching Strategy

Redis plays **two distinct roles** in this system and they are never
conflated:

| Role | Data | Durability | Source of truth? |
|---|---|---|---|
| **Reservation fast-path** | `hold:{showId}:{seatId}` | Ephemeral, TTL-bound | No — Postgres is authoritative (`07-concurrency-design.md`) |
| **Read-through cache** | Catalog: movies, shows, seatmaps | Ephemeral, TTL/invalidation-bound | No — Catalog Postgres is authoritative |

## Read-Through Cache (Catalog)

```
GET /shows/{id}/seatmap
        │
        ▼
   Redis GET seatmap:{showId}
        │
   HIT ─┴─ MISS
   │          │
 return    query Postgres, build seatmap
            (physical seats JOIN show_seat_inventory),
            SET seatmap:{showId} EX 10s,
            return
```

- **Key design**: `movie:{id}`, `show:{id}`, `seatmap:{showId}`,
  `venue:{id}`.
- **TTL**: short (5–15s) for `seatmap` — it reflects near-real-time
  inventory state and must not go badly stale; longer (minutes) for
  mostly-static `movie`/`venue` data.
- **Invalidation**: on any write to `show_seat_inventory` (hold placed,
  hold released, booking confirmed/cancelled), the Booking Service does
  **not** invalidate the Catalog cache directly (cross-service DB access
  is disallowed by design) — the seatmap TTL is intentionally short enough
  that the client is expected to re-fetch as part of the hold flow, and
  the hold/booking endpoints themselves always read live from Booking's
  own Postgres, never from this cache. The cache only ever backs the
  *browsing* read path, never the *write* decision path.
- **Failure behavior**: cache miss/Redis-down → fall through to Postgres
  read replica. Slower, not incorrect. Browsing degrades gracefully;
  booking correctness is entirely unaffected since the booking write path
  never trusts this cache.

## Rate Limiting

Redis is also used for a simple token-bucket / fixed-window rate limiter
(`INCR` + `EXPIRE`) on booking-critical endpoints
(`POST /bookings/holds`), keyed by `userId` and `IP`, to blunt bot-driven
hold-spam against hot shows. See `12-security-architecture.md`.

## What Redis Is Deliberately NOT Used For

- Not the system of record for any entity.
- Not used for cross-service data sharing (each service owns its own
  cache namespace/keys).
- Not used as a message queue — Kafka owns that role (see
  `09-event-driven-architecture.md`).
