# 06 — API Architecture

All APIs are REST over HTTPS, versioned via URL prefix (`/api/v1`), JSON
bodies. Internal service-to-service calls use the same REST contracts
(no separate gRPC layer in v1 — see trade-offs doc for why).

## Identity Service

```
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
```

## Catalog Service

```
GET    /api/v1/movies
GET    /api/v1/movies/{id}
GET    /api/v1/cities/{cityId}/venues
GET    /api/v1/shows?movieId=&venueId=&date=
GET    /api/v1/shows/{id}
GET    /api/v1/shows/{id}/seatmap        -- cached, read-through Redis

-- Admin (RBAC: ROLE_ADMIN / ROLE_VENUE_MANAGER)
POST   /api/v1/admin/movies
POST   /api/v1/admin/venues
POST   /api/v1/admin/venues/{id}/screens
POST   /api/v1/admin/screens/{id}/seats
POST   /api/v1/admin/shows
PATCH  /api/v1/admin/shows/{id}/pricing
```

## Booking Service

```
POST   /api/v1/bookings/holds
  Body: { showId, seatIds[] }
  Header: Idempotency-Key
  201 -> { holdId, expiresAt }
  409 -> { code: "SEAT_UNAVAILABLE", seatIds: [...] }   -- some/all requested seats are taken

POST   /api/v1/bookings
  Body: { holdId }
  Header: Idempotency-Key
  -- Creates a PENDING_PAYMENT booking bound to the hold, returns a
     paymentIntentId obtained from the Payment Service.
  201 -> { bookingId, status: "PENDING_PAYMENT", paymentIntentId }

GET    /api/v1/bookings/{id}
POST   /api/v1/bookings/{id}/cancel
```

## Payment Service

```
POST   /api/v1/payments/intents          -- called by Booking Service, not directly by client
  Body: { bookingId, amount, currency }
  Header: Idempotency-Key

POST   /api/v1/payments/webhook          -- called by external Payment Provider only
  Header: Signature (HMAC, provider-issued secret)
```

## Cross-Cutting API Rules

| Concern | Rule |
|---|---|
| **AuthN** | JWT bearer token, verified at the API Gateway; user identity propagated to services via a signed internal header (`X-User-Id`, `X-User-Roles`), not re-parsed per service. |
| **AuthZ** | RBAC checked per-endpoint (`ROLE_USER`, `ROLE_VENUE_MANAGER`, `ROLE_ADMIN`). Admin write endpoints require `ROLE_ADMIN` or scoped `ROLE_VENUE_MANAGER` for their own venue. |
| **Validation** | Request-level (Bean Validation) at the controller boundary; domain-level invariants enforced in the service/transaction layer, never trusted from the client alone. |
| **Idempotency** | Every state-mutating POST that could plausibly be retried (`holds`, `bookings`, `payments/intents`) requires an `Idempotency-Key` header. See `07-concurrency-design.md` / `11-reliability-and-failure-handling.md`. |
| **Error format** | Uniform problem-details style body: `{ code, message, details? }`, mapped to correct HTTP status (400 validation, 401/403 authz, 404 not found, 409 conflict — used specifically for seat contention, 429 rate-limited, 5xx). |
| **Sync vs async** | Everything above is synchronous request/response. Nothing the client is waiting on is ever "fire and forget" behind a 202 in v1 — the one exception is the payment **webhook**, which returns 200 immediately after durably recording the event and processes it via the outbox, so the provider is never blocked on our downstream work. |

## Why `409` for Seat Contention (not `423` or `400`)

A hold request that loses the race is a legitimate, expected outcome of
concurrent access to a shared resource — semantically a conflict, not a
malformed request or a locked-forever resource. Clients are expected to
handle `409` by refreshing the seat map, not by retrying the same request.
