# 01 — Scope and Responsibilities

## Canonical naming (used everywhere in this LLD)

`BookingController`, `BookingApplicationService`, `BookingDomainService`,
`SeatHoldDomainService`, `BookingRepository`, `IdempotencyService`,
`OutboxService`, `OutboxRelay`, `PaymentServiceClient`, `ShowServiceClient`,
`SeatHoldCoordinator` (Redis adapter).

No synonyms (e.g. "ReservationService", "BookingManager") are used anywhere
in code, docs, or diagrams.

## What the Booking Service owns

| Responsibility | Why Booking Service owns it | Why it doesn't belong elsewhere |
|---|---|---|
| Booking lifecycle & state transitions | The booking is the aggregate this service is named for; the state machine is its core business rule | Show Service knows nothing about a specific user's purchase attempt; Payment Service only knows about a payment, not a multi-seat booking |
| Seat claim / hold orchestration for a given show | A hold is a *booking-scoped* reservation ("this booking currently claims these seats"), not seat master data | Show Service owns seat *existence and layout*, not the transient act of claiming one for a purchase |
| Idempotency of booking commands | Idempotency is meaningful only in the context of the command being repeated (create/confirm/cancel booking) | A generic idempotency service would add a network hop for no benefit — the record is 1:1 with a booking command |
| Booking persistence & transaction boundaries | Only the owner of the aggregate can define correct transaction boundaries around it | N/A |
| Outbox events describing booking state changes | The transaction that changes booking state is the only place that can atomically also write the outbox row | Kafka producers cannot participate in the DB transaction |
| Booking-specific business rules (all-or-nothing seat acquisition, hold TTL, max seats per booking) | These rules only make sense relative to a booking | Show Service's rules are about venue/seat validity, not purchase policy |

## What the Booking Service explicitly does NOT own

### Auth Service
Not owned: authentication, credentials, JWT issuance/verification.
Booking Service **trusts** a verified `userId` injected by the API Gateway
(e.g. via a signed header / propagated principal) and does no credential
handling itself.

### Show Service
Owns: venues, shows, seat layout/master data, per-show pricing.
Booking Service **reads** (via `ShowServiceClient`) only what it needs to
create a valid booking:
- show existence/status (on sale, not cancelled)
- seat existence and price for the specific show
- (optionally) a short-lived cached view of "seat metadata," never seat
  *availability* — availability/ownership of a seat for a show is derived
  from Booking Service's own `booking_items` table, not from Show Service.

This is a deliberate boundary: **Show Service is the catalog. Booking
Service is the ledger of who currently claims a seat.** Mixing the two
would force Show Service to participate in booking transactions it has no
business reasoning about.

### Payment Service
Owns: payment processing, provider interaction, payment state machine,
webhook handling from the provider.
Booking Service **coordinates** with Payment Service (`PaymentServiceClient`)
by requesting a payment and consuming payment outcome callbacks, but never
talks to a payment provider directly and never stores card data.

### Notification Service
Owns: email/SMS/push delivery.
Booking Service only **emits domain events** (`BookingConfirmed`,
`BookingCancelled`, `BookingExpired`) via the outbox → Kafka. It has no
knowledge of channels, templates, or user contact preferences.

## Non-goals for this LLD
- No full LLD for Auth/Show/Payment/Notification/Analytics — only their
  contracts as seen from Booking Service.
- No CQRS, event sourcing, saga orchestration, or distributed transactions
  unless a concrete requirement in this document forces it (see
  `18-lld-tradeoffs.md` for why each was rejected).
