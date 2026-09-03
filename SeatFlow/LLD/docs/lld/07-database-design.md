# 07 — Database Design (PostgreSQL)

Four tables, deliberately not five — `seat_holds` is merged into
`booking_items` (see `02-domain-model.md`).

```sql
CREATE TABLE bookings (
    booking_id       UUID PRIMARY KEY,
    user_id          UUID NOT NULL,
    show_id          UUID NOT NULL,
    status           VARCHAR(20) NOT NULL,   -- HELD, PAYMENT_PENDING, CONFIRMED, CANCELLED, EXPIRED, FAILED
    total_amount     NUMERIC(12,2) NOT NULL,
    currency         CHAR(3) NOT NULL,
    held_until       TIMESTAMPTZ NOT NULL,
    version          BIGINT NOT NULL DEFAULT 0,   -- optimistic lock for status transitions
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bookings_user_id ON bookings(user_id);
CREATE INDEX idx_bookings_status_held_until ON bookings(status, held_until)
    WHERE status IN ('HELD','FAILED');   -- fast scan for the expiry worker

CREATE TABLE booking_items (
    booking_item_id  UUID PRIMARY KEY,
    booking_id       UUID NOT NULL REFERENCES bookings(booking_id),
    show_id          UUID NOT NULL,
    seat_id          UUID NOT NULL,
    price_at_hold    NUMERIC(12,2) NOT NULL,
    status           VARCHAR(20) NOT NULL,   -- HELD, CONFIRMED, RELEASED, CANCELLED
    hold_token       UUID NOT NULL,
    held_until       TIMESTAMPTZ NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_booking_items_booking_id ON booking_items(booking_id);

-- THE core-invariant enforcement: at most one ACTIVE claim per seat per show
CREATE UNIQUE INDEX uq_active_seat_claim
    ON booking_items(show_id, seat_id)
    WHERE status IN ('HELD','CONFIRMED');

CREATE TABLE idempotency_records (
    idempotency_key      VARCHAR(200) NOT NULL,
    user_id              UUID NOT NULL,
    operation             VARCHAR(50) NOT NULL,   -- e.g. CREATE_BOOKING
    request_fingerprint  VARCHAR(64) NOT NULL,    -- sha256 of normalized request body
    status                VARCHAR(20) NOT NULL,   -- NEW, IN_PROGRESS, COMPLETED, FAILED
    response_snapshot    JSONB,
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at           TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (idempotency_key, user_id, operation)
);

CREATE TABLE outbox_events (
    event_id       UUID PRIMARY KEY,
    aggregate_id   UUID NOT NULL,          -- booking_id
    event_type     VARCHAR(50) NOT NULL,   -- BookingHeld, BookingConfirmed, BookingCancelled, BookingExpired, BookingFailed
    payload        JSONB NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at   TIMESTAMPTZ,
    retry_count    INT NOT NULL DEFAULT 0
);
CREATE INDEX idx_outbox_unpublished ON outbox_events(created_at)
    WHERE published_at IS NULL;   -- fast scan for OutboxRelay
```

## How the invariant is actually guaranteed

`uq_active_seat_claim` is a **partial unique index**, not a business rule
in Java. A concurrent `INSERT` for the same `(show_id, seat_id)` while one
is already `HELD`/`CONFIRMED` raises `SQLState 23505`, which
`BookingRepository` translates into a domain `SeatUnavailableException`.
Releasing a seat (`RELEASED`/`CANCELLED`) removes it from the index's
predicate automatically — no explicit "delete and reinsert" dance needed.

## Isolation level

`READ COMMITTED` (Postgres default) is sufficient. The unique index does
the work that would otherwise require `SERIALIZABLE`; we don't pay for
serializable's higher abort rate for a guarantee we already have more
cheaply. This is verified in `16-testing-strategy.md`'s concurrency test.
