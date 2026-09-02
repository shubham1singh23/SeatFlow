# 05 — Data Architecture

## Database Choice

| Criterion | PostgreSQL | MySQL | MongoDB |
|---|---|---|---|
| ACID multi-row transactions | Yes | Yes | Limited (single-doc strong, multi-doc weaker ergonomics) |
| Unique constraints as a correctness backstop | Yes, native, well-understood | Yes | Possible but less idiomatic for this workload |
| Relational integrity (seat↔screen↔show↔booking) | Strong | Strong | Requires manual enforcement |
| Partial/expression indexes (needed for "one CONFIRMED booking per seat per show") | Yes | No | No |
| Read replicas / mature ecosystem | Yes | Yes | Yes |

### Decision

**PostgreSQL**, one database per service (Identity, Catalog, Booking,
Payment). Rejected MongoDB: the core guarantee of this system — "at most
one CONFIRMED booking per (show_id, seat_id)" — is most naturally and
cheaply enforced as a **partial unique index**, which is a first-class
relational feature. Modeling that safely in a document store would mean
re-implementing a mini-lock/constraint layer ourselves, which is exactly
the kind of accidental complexity this project avoids. MySQL is a viable
alternative to Postgres; Postgres is chosen for partial indexes, `SELECT …
FOR UPDATE SKIP LOCKED` support, and richer transaction isolation controls,
all used directly in `07-concurrency-design.md`.

## Seat Inventory Model

```
Venue
  └── Screen
        └── Seat (physical: row, number, category)   [Catalog Service, static]

Show (a screening of a Movie on a Screen at a time)     [Catalog Service]
  └── ShowSeatInventory (per show, per physical seat)   [Booking Service]
        state: AVAILABLE | HELD | BOOKED
```

**Physical seats and show-specific availability are deliberately separate
concepts, owned by different services:**

- **Catalog Service** owns the *physical* seat layout (row/number/category)
  — this rarely changes and is shared across every show on that screen.
- **Booking Service** owns *`ShowSeatInventory`* — one row per (show,
  seat), created when a show is published, representing that seat's
  booking state **for that specific show only**. A seat can be AVAILABLE
  for the 3pm show and BOOKED for the 7pm show simultaneously; they are
  independent rows.

This separation means Catalog outages never block Booking's core write
path (Booking already has its own copy of what it needs), and Booking's
write-heavy, lock-sensitive table is never polluted with rarely-changing
Catalog data.

### Source of Truth

| Concept | Source of truth | Notes |
|---|---|---|
| A seat physically exists | Catalog `seats` table | Static |
| A seat is AVAILABLE / HELD / BOOKED for show X | Booking `show_seat_inventory` (Postgres) | **The** source of truth |
| A seat is *currently* held (fast-path check) | Redis key `hold:{showId}:{seatId}` | Cache/optimization only — TTL-based, can expire; Postgres is authoritative if it disagrees |
| A booking is CONFIRMED | Booking `bookings` table, guarded by partial unique index | Durable, authoritative |
| A payment succeeded | Payment `payments` table | Durable, authoritative; Booking only learns this via event |

## Core Schema (simplified)

```sql
-- Booking Service DB
CREATE TABLE show_seat_inventory (
  show_id     BIGINT NOT NULL,
  seat_id     BIGINT NOT NULL,
  status      TEXT NOT NULL CHECK (status IN ('AVAILABLE','HELD','BOOKED')),
  hold_id     UUID,
  version     INT NOT NULL DEFAULT 0,          -- optimistic concurrency
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (show_id, seat_id)
);

CREATE TABLE holds (
  hold_id     UUID PRIMARY KEY,
  show_id     BIGINT NOT NULL,
  seat_id     BIGINT NOT NULL,
  user_id     BIGINT NOT NULL,
  expires_at  TIMESTAMPTZ NOT NULL,
  status      TEXT NOT NULL CHECK (status IN ('ACTIVE','CONSUMED','EXPIRED'))
);

CREATE TABLE bookings (
  booking_id      UUID PRIMARY KEY,
  show_id         BIGINT NOT NULL,
  user_id         BIGINT NOT NULL,
  status          TEXT NOT NULL CHECK (status IN ('PENDING_PAYMENT','CONFIRMED','CANCELLED','FAILED')),
  idempotency_key TEXT NOT NULL UNIQUE,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE booking_seats (
  booking_id BIGINT REFERENCES bookings(booking_id),
  show_id    BIGINT NOT NULL,
  seat_id    BIGINT NOT NULL
);

-- THE correctness backstop: at most one CONFIRMED booking per seat per show
CREATE UNIQUE INDEX uq_confirmed_seat
  ON booking_seats (show_id, seat_id)
  WHERE EXISTS (
    SELECT 1 FROM bookings b
    WHERE b.booking_id = booking_seats.booking_id AND b.status = 'CONFIRMED'
  );
-- (Implemented in practice via a status column on booking_seats + partial
--  unique index on (show_id, seat_id) WHERE status = 'CONFIRMED' — see
--  07-concurrency-design.md for the exact enforced form.)

CREATE TABLE booking_outbox (
  id          BIGSERIAL PRIMARY KEY,
  event_type  TEXT NOT NULL,
  payload     JSONB NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  published   BOOLEAN NOT NULL DEFAULT false
);
```

## Read/Write Pattern Summary

| Table | Pattern | Scaling approach |
|---|---|---|
| Catalog (`movies`,`venues`,`shows`) | Read-heavy, write-light | Read replicas + Redis cache |
| `show_seat_inventory` | Write-heavy on hot shows, small row count per show | Partitioned by `show_id` conceptually; hot rows handled by concurrency design, not sharding |
| `bookings` / `payments` | Append-heavy, time-ordered | Range-partitioned by `created_at` (monthly), archived after N months |

## Partitioning & Archival

`bookings` and `payments` are the fastest-growing tables (see scale
estimation: ~440M rows/year each). Both are **range-partitioned by month**
on `created_at`. Partitions older than the active retention window (e.g.
18 months) are detached and archived to cold storage — this is called out
as a **future implementation** concern, not built in v1, but the schema is
partition-ready from day one so it doesn't require a painful migration
later.
