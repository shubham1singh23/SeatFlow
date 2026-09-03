# 12 — Failure Handling Matrix

| Failure | Expected behavior |
|---|---|
| Redis unavailable | `SeatHoldCoordinator` catches the exception and returns "not fast-path-claimed" (never "claimed"), so every request falls through to the Postgres unique-constraint check. Correctness preserved; latency/DB load increases. No circuit breaker needed to *protect correctness* — it exists only to protect latency (fail fast rather than wait on Redis timeout repeatedly) |
| PostgreSQL unavailable | Booking Service cannot create/confirm/cancel bookings — returns `503`. This is the one dependency with no safe degraded mode, because it is the source of truth; documented as a hard dependency in the README |
| Show Service unavailable | `CreateBooking` cannot validate seats → `503`/`424 Failed Dependency`. No stale-cache fallback for seat *existence* (would risk booking a seat that no longer exists); short-TTL cache is acceptable only for read-heavy, non-authoritative metadata (e.g. show title for display), not for the validation gate |
| Payment Service unavailable | `ConfirmBooking` (initiate payment) returns `503`; booking remains `HELD` and can be retried within the hold TTL |
| Payment timeout | See `10-payment-interaction.md` — status stays `PAYMENT_PENDING`, resolved by callback or reconciliation sweep, never assumed failed |
| Payment succeeds but response lost | Reconciliation sweep queries Payment Service for ground truth |
| Booking Service crashes (any point) | Nothing is lost: DB transactions are atomic; anything not committed simply didn't happen and the client's retry (with the same Idempotency-Key) resumes safely |
| Kafka unavailable | `OutboxRelay` keeps rows unpublished, retries with backoff on next sweep; booking transactions are unaffected (outbox write is local Postgres, not Kafka) |
| Duplicate request | Resolved by `IdempotencyService` — see `09-idempotency.md` |
| Duplicate payment webhook | No-op 200, resolved by `paymentId` check — see `10-payment-interaction.md` |
| Hold expires during payment | Cannot happen by construction — expiry worker excludes `PAYMENT_PENDING` — see `03-booking-lifecycle.md` |
| Two users request same seat | Exactly one `INSERT` succeeds (unique index); the other gets `409 SEAT_UNAVAILABLE` — see `08-concurrency-and-seat-locking.md` |
| Partial multi-seat hold failure | Whole booking fails, whole transaction rolls back — see `08-...md` "all-or-nothing" |
| Outbox relay crashes | Unpublished rows remain unpublished; next relay instance/restart picks them up (idempotent republish is safe — consumers dedupe on `event_id`) |
| Consumer receives duplicate event | Not Booking Service's problem to solve, but its contract enables it: `event_id` is stable and included for consumer-side dedup |

## Decision key used above
Each row above resolves to one of: **retry** (client or scheduled),
**fail fast** (503/409, no silent success), **fall through to source of
truth** (Redis→Postgres, sync response→reconciliation sweep), or
**no-op idempotent success** — never a bare "retry" with no further
detail, per the brief's explicit instruction.
