# 13 — Scalability Strategy

## Stateless Services, Horizontal Scaling

Every application service (Identity, Catalog, Booking, Payment,
Notification) is stateless — no in-memory session/seat state — so scaling
is "add instances behind the load balancer / gateway," standard for all
of them. This is necessary but **not sufficient** for the booking path;
see below.

## Where Horizontal Scaling Does *Not* Help: Hot Seats

Adding more Booking Service instances increases the system's capacity to
handle *many different* seats/shows concurrently. It does **not** help
when 30 instances are all trying to hold the *same* seat — that contention
is resolved at the data layer (Redis single-key atomicity + Postgres
conditional update), not by adding compute. This is precisely why
`07-concurrency-design.md` treats "more app instances" and "correct
concurrency control" as two separate problems.

## Bottleneck Analysis

| Layer | Bottleneck under load | Mitigation |
|---|---|---|
| API Gateway / LB | Connection/request throughput | Horizontal scale, standard |
| Catalog Service | Read QPS on hot shows during on-sale moments | Redis read-through cache absorbs the vast majority of reads; Postgres read replicas for cache misses |
| Booking Service (app tier) | CPU/connections across *many* concurrent shows | Horizontal scale (stateless) |
| Booking Service (data tier) | Row-level contention on **one hot seat/show** | Redis admission filter (07) narrows this to one fast DB transaction per seat; not solved by scaling app tier |
| Postgres (Booking DB) connections | Connection pool exhaustion under peak | Bounded pool + PgBouncer-style pooling; short transaction hold times (07) keep connections turning over fast |
| Kafka | Broker/partition throughput | Partition count set generously above estimated peak event rate (`02-scale-estimation.md`); consumer groups scale independently |
| Notification dispatch | External provider rate limits | Consumer-side throttling, independent of booking traffic shape |

## 1x / 10x / 100x Traffic

| | 1x (baseline) | 10x | 100x |
|---|---|---|---|
| **Stateless services** | Baseline instance count | Scale out linearly | Scale out linearly; eventually gateway/LB tier itself needs multi-region consideration |
| **Catalog reads** | Cache handles most reads | Cache hit ratio remains the dominant factor — still fine | May need cache tier sharding / dedicated Redis cluster per hot region |
| **Booking hot-seat contention** | Handled by Redis+Postgres hybrid comfortably | Still handled — Redis admission filter is what prevents this tier from becoming the bottleneck first | A single hot show's seat count (~150) caps the *maximum useful* contention on any one key regardless of total traffic — the mechanism doesn't degrade further because the hot-key set size is bounded by physical seat count, not by request volume |
| **Postgres (Booking)** | Single primary + replica(s) sufficient | Read replicas for booking-history reads; primary still fine for the narrow write path | Sharding by `show_id` range becomes the natural next step if a single primary's write throughput is ever actually the ceiling — noted as **future evolution**, not built |
| **Kafka** | Comfortably under capacity (`02`) | Comfortably under capacity | Increase partition count / broker count; throughput headroom was intentionally generous from the start |

## Why the Hot-Seat Problem Doesn't Get Worse at 100x

This is worth stating explicitly because it's a common misconception: total
system traffic can grow 100x while the *worst-case contention on a single
seat* stays roughly the same, because that ceiling is bounded by "how many
people want seat A12 for show X," which is bounded by the number of people
who could plausibly be racing for one physical seat in one moment — not by
overall platform QPS. This is why the concurrency design in `07` is sized
around **contention per key**, not overall throughput.
