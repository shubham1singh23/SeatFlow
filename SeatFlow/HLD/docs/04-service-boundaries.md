# 04 — Service Boundaries

For each service: capability, data owned, callers, dependencies, failure
blast radius, and scaling profile.

## Identity Service

- **Capability**: registration, login, token issuance/refresh, password
  hashing.
- **Owns**: `users`, `credentials`, `refresh_tokens`.
- **Called by**: API Gateway (token validation can be local JWT verify —
  see `12-security-architecture.md`), all services indirectly via JWT claims.
- **Depends on**: its own Postgres only.
- **If it fails**: new logins/registrations fail; *already-issued* JWTs
  remain valid until expiry (stateless verification), so existing sessions
  keep working — booking/browsing is not immediately impacted.
- **Scaling**: stateless, horizontally scaled behind the gateway; low QPS
  relative to Catalog/Booking.

## Catalog Service

- **Capability**: movie/event catalog, venues, screens, physical seat
  layout, shows, pricing configuration.
- **Owns**: `movies`, `venues`, `screens`, `seats` (physical, per screen),
  `shows`, `show_pricing`.
- **Called by**: Clients (browse/search), Booking Service (to validate a
  show/seat exists and fetch price at hold-time).
- **Depends on**: its own Postgres; Redis for read-through caching.
- **If it fails**: browsing/search degrades to cached data (Redis) or
  fails gracefully; **existing** holds/bookings in Booking Service are
  unaffected since Booking snapshots price and seat identity at hold time.
- **Scaling**: read-heavy, horizontally scaled, cache-first. This is the
  service optimized purely for **availability**.

## Booking Service (correctness-critical)

- **Capability**: per-show seat state machine, seat holds, booking
  creation/confirmation/cancellation.
- **Owns**: `seat_inventory` (per show, per seat: AVAILABLE / HELD /
  BOOKED), `holds`, `bookings`, `booking_outbox`.
- **Called by**: Clients (hold/confirm/cancel), Payment Service (via events,
  to confirm/release on payment result).
- **Depends on**: its own Postgres (source of truth), Redis (fast-path
  reservation, not source of truth), Kafka (outbox publish).
- **If it fails**: booking is unavailable — by design, this is the one
  acceptable "hard failure" surface in the system, because a degraded
  booking write path is safer than a booking path that might double-sell a
  seat. Reads of *existing* bookings can still be served from Postgres
  replicas even during a partial outage.
- **Scaling**: horizontally scaled stateless app tier; the actual
  concurrency bottleneck is at the **key** level (one seat, one show), not
  the service level — see `07-concurrency-design.md` for why more
  instances alone don't solve the hot-seat problem.

## Payment Service

- **Capability**: payment intent creation, provider integration, webhook
  verification/processing, refunds.
- **Owns**: `payment_intents`, `payments`, `payment_outbox`.
- **Called by**: Booking Service (create intent), Payment Provider
  (webhook).
- **Depends on**: its own Postgres, external Payment Provider, Kafka.
- **If it fails**: users cannot pay for new holds (holds simply expire via
  TTL — no inventory is stuck); existing confirmed bookings are unaffected.
- **Scaling**: stateless; webhook endpoint must be idempotent and
  low-latency to satisfy provider retry SLAs.

## Notification Service

- **Capability**: consumes booking/payment events, sends email/SMS/push.
- **Owns**: `notification_log` (delivery attempts, for idempotency/audit).
- **Called by**: nobody synchronously — pure Kafka consumer.
- **Depends on**: Kafka, external Notification Provider.
- **If it fails**: users don't get notified promptly; bookings and
  payments are entirely unaffected. This is the system's deliberately
  "least important" failure domain — it is isolated so its failures
  **cannot** propagate anywhere else.
- **Scaling**: consumer-group based horizontal scaling, independent of
  booking traffic shape.

## Cross-Service Rule

No service ever reads another service's database directly. All
cross-service data access is either:
1. A synchronous REST call to the owning service, or
2. An asynchronous event the owning service published.

This is what makes the data-ownership diagram in
`08-database-architecture.drawio` a clean tree instead of a shared mess.
