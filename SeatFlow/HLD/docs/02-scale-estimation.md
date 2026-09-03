# 02 — Scale Estimation

> All numbers below are **design assumptions** used to drive architecture
> decisions (partitioning, cache sizing, connection pools, Kafka
> partitions). They are not measured production statistics.

## Base Assumptions

| Metric | Assumption |
|---|---|
| Registered users | 5,000,000 |
| Daily active users (DAU) | 300,000 |
| Cities | 50 |
| Venues | 2,000 |
| Screens | 6,000 (avg 3/venue) |
| Avg seats/screen | 150 |
| Shows/day | 30,000 (5 shows/screen/day) |
| Avg bookings/show | 40 (avg ~27% fill rate blended across normal + hot shows) |
| Bookings/day | ~1,200,000 |
| Search/browse requests per booking | ~15:1 (typical browse-to-buy ratio) |

## Traffic Estimation

```
Average booking QPS   = 1,200,000 / 86,400s      ≈ 14 QPS
Average search QPS    = 14 * 15                   ≈ 210 QPS
Peak multiplier (evening/weekend) ≈ 8x average
Peak booking QPS       ≈ 112 QPS
Peak search QPS        ≈ 1,680 QPS
```

### Blockbuster / Hot-Show Spike (design assumption, not average traffic)

A single blockbuster release can concentrate demand on a handful of shows:

```
Hot show: 150 seats, ticket sale opens at T0
Assumption: 5,000 concurrent hold attempts within first 60s across ~150 seats
=> per-seat contention: up to ~30 simultaneous requests for the *same* seat
```

This single scenario is why seat-level concurrency, not just service-level
throughput, is the design driver (see `07-concurrency-design.md`). No
amount of horizontal scaling of stateless services fixes contention on one
hot key — it must be solved at the data/lock layer.

## Storage Estimation

| Entity | Row size (approx) | Volume | Approx size |
|---|---|---|---|
| Users | 0.5 KB | 5M | 2.5 GB |
| Movies/Events | 2 KB | 50,000 | 100 MB |
| Venues/Screens/Seats | 0.2 KB | ~1M seat rows (6,000 screens × ~150) | 200 MB |
| Shows | 0.3 KB | 30,000/day → ~11M/year | ~3.3 GB/year |
| Bookings | 1 KB | 1.2M/day → ~440M/year | ~440 GB/year |
| Payments | 0.5 KB | 1.2M/day → ~440M/year | ~220 GB/year |

**Implication:** Bookings/Payments are the fast-growing tables and are the
primary candidates for time-based partitioning and archival (see
`05-data-architecture.md`).

## Event Volume (Kafka)

Each booking produces ~4–6 domain events (Created, PaymentInitiated,
PaymentSucceeded/Failed, Confirmed/Cancelled).

```
Avg event rate  ≈ 14 bookings/s * 5 events        ≈ 70 events/s
Peak event rate ≈ 112 bookings/s * 5 events        ≈ 560 events/s
```

This is comfortably within a single reasonably-sized Kafka cluster; the
architectural driver for Kafka is **decoupling and reliability**, not raw
throughput (see `09-event-driven-architecture.md`).

## Cache Sizing (Redis)

- **Seat holds**: worst case, every seat in every currently-on-sale show is
  held simultaneously. With ~30,000 shows/day × 150 seats × small key
  payload (~150 bytes) ≈ 675 MB if *all* seats across *all* shows were held
  at once — a deliberately pessimistic upper bound. Realistic working set
  (shows actively selling in the current window) is a small fraction of
  this.
- **Catalog cache** (movies, shows, seat maps): read-heavy, TTL-based,
  low-tens-of-MB working set for actively browsed cities.

## Summary of Architectural Drivers From Scale

1. Absolute QPS is modest — this is **not** a raw-throughput problem.
2. The problem is **contention concentration**: many requests racing for a
   *few* keys (hot seats), not evenly distributed load.
3. Storage growth is dominated by Bookings/Payments → partition & archive.
4. Kafka is sized generously by throughput; its value here is reliability
   (outbox, retries, DLQ), not scale.
