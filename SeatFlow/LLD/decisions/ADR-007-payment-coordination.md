# ADR-007: Payment Coordination — Async Callback + Reconciliation, No Saga

## Context
Payment provider round-trips can take seconds to minutes (3-D Secure,
bank redirects); Booking Service must coordinate without owning payment
processing.

## Problem
How does Booking Service learn the outcome of a payment reliably,
including when the callback is lost, duplicated, or arrives out of
expected order — without a full saga orchestrator?

## Options considered
- Synchronous blocking call held open until payment resolves — rejected,
  unworkable latency, ties up resources.
- Saga orchestration with compensating transactions across services —
  rejected as unnecessary for a two-party (Booking, Payment) interaction
  with no multi-service compensation chain to coordinate.
- Async callback (webhook) from Payment Service, idempotent on
  `paymentId`, backed by a scheduled reconciliation sweep for stuck
  `PAYMENT_PENDING` bookings.

## Decision
Async callback + reconciliation sweep, detailed case-by-case in
`docs/lld/10-payment-interaction.md`.

## Why we chose it
It handles every distributed-failure case in §14 of the original brief
(timeout, lost response, crash-before-persist, duplicate webhook,
out-of-order webhook, late success after expiry) without inventing a
generalized orchestration framework for what is, structurally, a single
request/callback relationship.

## Trade-offs
Requires a scheduled reconciliation job as a backstop (extra operational
component) rather than relying purely on the callback.

## Consequences
`PAYMENT_PENDING` is explicitly excluded from the hold-expiry worker's
eligibility (`docs/lld/03-booking-lifecycle.md`) — this exclusion is load
bearing and must not be "simplified away" by a future change.

## Failure scenarios
Full matrix in `docs/lld/10-payment-interaction.md`.

## Future evolution
If SeatFlow adds multi-service checkout (bundled products, loyalty
points) requiring true cross-service compensation, a saga becomes
justified — not before.
