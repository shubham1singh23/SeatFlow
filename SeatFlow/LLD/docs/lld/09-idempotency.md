# 09 — Idempotency

## Where required
`POST /bookings` (create), `POST /bookings/{id}/confirm`,
`POST /bookings/{id}/cancel`, and the Payment Service webhook consumer
(keyed on `paymentId`, not a client-supplied key — see
`10-payment-interaction.md`).

## Design

Client sends `Idempotency-Key: <opaque string>` (scoped per `userId` +
operation — see PK in `07-database-design.md`, so the same key value used
for a different operation or by a different user is a different record).

```
Idempotency-Key + userId + operation
            |
            v
   IdempotencyService.begin(key, operation, requestFingerprint)
            |
   +--------+----------------------------------+
   |                                            |
 no existing row                       existing row found
   |                                            |
   v                                            v
INSERT (status=IN_PROGRESS)          fingerprint matches?
   | (unique PK enforces               /            \
   |  "only one writer wins")        yes              no
   v                                   |                |
 proceed with the command      status?             409 Conflict
   |                          /    |    \        (same key reused
   v                 IN_PROGRESS COMPLETED FAILED  for a different
 on success:              |         |        |     request body)
 store response_snapshot, 409       return   allow
 status=COMPLETED     "retry        stored   retry
   |                   later"       response
 on failure:
 status=FAILED (caller may retry with same key)
```

## Concurrent duplicate requests (the hard case)

Two identical requests race in at the same instant. Both call
`IdempotencyService.begin`, which does an `INSERT ... ON CONFLICT DO
NOTHING` against the `(idempotency_key, user_id, operation)` primary key.
Exactly one insert succeeds; the loser's `INSERT` returns zero rows, so it
falls into the "existing row found, status=IN_PROGRESS" branch and returns
`409 Retry-After: short-backoff` rather than double-executing the command.
This mirrors the seat-claim mechanism: **the database's own uniqueness
guarantee resolves the race, not application logic**, so it is safe with
any number of Booking Service instances.

## Key format & TTL
`idempotency_key`: client-generated UUID or ULID, max 200 chars.
`expires_at`: `created_at + 24h` — long enough to cover realistic client
retry windows (mobile app backgrounding, payment redirect flows) without
keeping the table unbounded; a daily cleanup job deletes expired rows.

## Same key, different payload
Rejected with `422 Unprocessable Entity` (fingerprint mismatch) rather
than silently executing the new payload — protects against a client bug
reusing a key across genuinely different requests.
