# 14 — Security (Booking Service scope only)

- **Authentication propagation**: Booking Service trusts a verified
  principal (`userId`) forwarded by the API Gateway (e.g. signed JWT
  validated upstream, or an internal header set only by the gateway on a
  trusted network path). Booking Service does not verify passwords or
  issue tokens.
- **Authorization**: every read/mutate on a booking checks
  `booking.userId == authenticatedUserId` (or an explicit admin scope) in
  `BookingApplicationService` before touching domain state — never left to
  the repository layer or assumed from routing.
- **Preventing cross-user booking access**: `GET/POST .../{bookingId}/*`
  return `404` (not `403`) to a non-owner to avoid confirming a
  booking's existence to someone probing IDs.
- **Input validation**: `seatIds` bounded (max per booking, e.g. 8, to
  bound worst-case transaction size), `showId`/`seatId` validated as
  well-formed UUIDs before any DB/Redis call, request size limits at the
  gateway.
- **API abuse**: rate limiting on `POST /bookings` per user (prevents one
  user from spamming hold attempts to grief seat availability) — enforced
  at the gateway, Booking Service adds a secondary per-user in-flight-hold
  cap as defense in depth (e.g. max 3 concurrent `HELD` bookings per user).
- **Idempotency-key abuse**: keys are scoped per `(userId, operation)`, so
  a key cannot be replayed across users to hijack another user's booking
  (`09-idempotency.md`'s PK design already enforces this).
- **Sensitive data**: no card data ever stored — `paymentMethodToken` is
  opaque and forwarded, never logged in full (see next doc).
- **Secrets**: DB/Redis/Kafka credentials via the platform's secret
  manager, injected as environment variables, never in source or config
  files committed to the repo.
