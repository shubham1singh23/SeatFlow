# 06 — API Design

All endpoints require an authenticated principal (propagated from API
Gateway/Auth). All mutating endpoints require `Idempotency-Key`.

### `POST /api/v1/bookings`
- **Purpose**: create a booking = claim seats (state `HELD`)
- **Request**: `{ showId, seatIds: [...] }`
- **Response `201`**: `BookingResponseDto { bookingId, status, items[],
  totalAmount, heldUntil }`
- **Errors**: `409 SEAT_UNAVAILABLE` (any seat already claimed —
  all-or-nothing), `404 SHOW_NOT_FOUND`, `422` (invalid seat for show),
  `409 IDEMPOTENCY_IN_PROGRESS`, `422 IDEMPOTENCY_FINGERPRINT_MISMATCH`
- **Idempotency**: required. **Concurrency implication**: this is the
  endpoint the whole `08-concurrency-and-seat-locking.md` design protects.

### `GET /api/v1/bookings/{bookingId}`
- **Purpose**: read booking detail
- **AuthZ**: caller must be the booking's `userId` (or admin scope)
- **Response `200`**: `BookingResponseDto`
- **Errors**: `404`, `403` (exists but not yours — never leak existence
  details beyond a generic 403/404 choice; we use `404` to avoid
  confirming existence to a non-owner)

### `POST /api/v1/bookings/{bookingId}/confirm`
- **Purpose**: user-initiated "proceed to payment" — transitions
  `HELD → PAYMENT_PENDING` and calls `PaymentServiceClient.initiatePayment`
- **Request**: `{ paymentMethodToken }` (opaque, never raw card data)
- **Response `202 Accepted`**: `{ bookingId, status: PAYMENT_PENDING,
  paymentRedirectUrl? }` — 202 because payment outcome is async
- **Errors**: `409 INVALID_STATE` (not HELD), `410 HOLD_EXPIRED`
- **Idempotency**: required (prevents double payment initiation on client
  retry)

*(Note: actual `CONFIRMED` transition happens via the internal payment
callback endpoint below, not this one — this endpoint only starts
payment.)*

### `POST /internal/v1/bookings/{bookingId}/payment-callback`
- **Purpose**: Payment Service → Booking Service outcome delivery
  (internal, service-to-service auth, not user-facing)
- **Request**: `{ paymentId, outcome: SUCCEEDED|FAILED, ... }`
- **Idempotency**: keyed on `paymentId`, idempotent by construction (see
  `10-payment-interaction.md`) — safe to call any number of times

### `POST /api/v1/bookings/{bookingId}/cancel`
- **Purpose**: user cancels a HELD or CONFIRMED booking (policy-gated for
  the latter)
- **Response `200`**: updated `BookingResponseDto`
- **Errors**: `409 INVALID_STATE`, `403` (not owner)
- **Idempotency**: required (retry-safe — cancelling an already-cancelled
  booking is a no-op 200, not an error)

### `POST /internal/v1/bookings/expire` — *not exposed*, no HTTP endpoint
Expiration is driven entirely by the internal scheduled worker
(`@Scheduled` job calling `BookingApplicationService.expireDueHolds()`),
not an externally triggerable API — exposing it would let a client force
another user's hold to expire early.

## DTOs, not entities
`BookingResponseDto`, `CreateBookingRequestDto`, `ConfirmBookingRequestDto`
are distinct classes in `api/dto`, mapped via `BookingMapper` (MapStruct or
hand-written) — `Booking`/`BookingItem` JPA entities never serialize
directly to JSON (avoids leaking `holdToken`, internal `version`, etc.).
