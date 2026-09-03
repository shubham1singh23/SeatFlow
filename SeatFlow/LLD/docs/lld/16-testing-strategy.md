# 16 — Testing Strategy

## Unit tests (no Spring context, no DB)
- `BookingDomainService`: every legal/illegal transition in
  `03-booking-lifecycle.md`'s table, `totalAmount` computation, guard
  rejection for non-owner.
- `SeatHoldDomainService`: all-or-nothing behavior with a fake
  `SeatHoldCoordinator`/`BookingRepository`.
- `IdempotencyService`: fingerprint-mismatch rejection, status-branch
  logic.

## Integration tests (Testcontainers: real Postgres + real Redis)
- `BookingRepository`: unique-index violation actually raised and mapped
  to `SeatUnavailableException`.
- Full `@Transactional` boundary tests: forced failure mid-transaction
  rolls back both `bookings` and `outbox_events` writes together (proves
  the outbox's atomicity claim, not just asserts it in prose).
- `OutboxRelay` against a real (test) Kafka (Testcontainers Kafka module):
  publish, mark published, resume-after-crash simulation (kill relay
  mid-batch, restart, verify no gaps and acceptable duplication).

## Concurrency tests — the ones that actually demonstrate the invariant
```
Setup: one show, one seat, N=100 concurrent booking attempts
        (real threads/virtual threads hitting a real Testcontainers Postgres,
         ideally also 2+ Booking Service instances against the same DB/Redis
         to prove multi-instance safety, not just multi-thread-in-one-JVM)
Assert: exactly 1 booking reaches HELD for that seat,
        99 receive SEAT_UNAVAILABLE (409),
        zero receive a false "success"
```
Additional concurrency scenarios: duplicate idempotent request fired N
times concurrently → exactly one executes, N-1 return the stored response;
simulated Redis outage mid-test → invariant still holds (proves the
fail-open-to-DB behavior in `12-error-and-failure-handling.md`).

## What is explicitly NOT claimed
These are **designed and (where noted) implementable** test scenarios.
This document does not claim they have been run against production
traffic, nor does it fabricate benchmark numbers — see
`18-lld-tradeoffs.md`'s closing note on this.
