# 01 — Requirements

## Problem Statement

SeatFlow is a distributed ticket booking platform for movies and events. The
central engineering problem is **safe concurrent seat allocation**: when N
users race for the same seat on the same show, exactly one must win, under
every realistic failure mode (crashes, retries, duplicate requests, partial
outages of Redis/Kafka/Postgres, payment/booking split-brain).

Everything else in this system — service boundaries, data ownership,
caching, eventing — is designed in service of that guarantee, not the other
way around.

## Actors

| Actor | Description |
|---|---|
| Customer | Browses, books, pays, cancels, views history |
| Admin / Venue Manager | Manages movies, venues, screens, shows, pricing |
| Payment Provider | External, webhook-driven |
| Notification Provider | External (email/SMS/push gateway) |

## Functional Requirements

### Customer
- Register / login (JWT-based session)
- Search movies/events, browse cities and venues
- View show schedules and real-time seat availability
- Hold seats temporarily (checkout intent)
- Pay for a hold and receive a confirmed booking
- Cancel an eligible booking (refund workflow)
- View booking history
- Receive booking/payment notifications

### Admin / Venue Manager
- CRUD movies/events
- CRUD venues, screens, seat layouts
- Create/modify shows and schedules
- Configure pricing per show/seat-category
- Monitor bookings and inventory for a show

## Non-Functional Requirements

| Property | Requirement | Rationale |
|---|---|---|
| **Consistency** | Strong consistency for seat allocation & payment state. Eventual consistency acceptable for notifications, search index, analytics. | Double-booking or double-charging is a business-breaking bug; a delayed email is not. |
| **Availability** | Browsing/search should degrade gracefully and stay available even if booking is impaired. | Users should always be able to look, even during a booking-path incident. |
| **Performance** | p99 read (search/seat map) target: low hundreds of ms. p99 seat-hold decision target: low hundreds of ms under contention. *(Design targets, not measured benchmarks.)* | Booking is on the interactive critical path; users abandon slow flows. |
| **Scalability** | All application services must be stateless and horizontally scalable. Hot shows (blockbuster releases) must not degrade unrelated shows. | Traffic is extremely spiky and skewed to a small number of "hot" shows. |
| **Reliability** | No single downstream failure (Redis, Kafka, one Postgres instance) should silently produce an incorrect booking or a lost payment. | Money and inventory correctness must survive partial outages. |
| **Security** | AuthN/AuthZ on all mutating endpoints, RBAC for admin actions, verified payment webhooks, rate limiting on booking-critical endpoints. | Booking endpoints are natural targets for abuse (bots, scalping, hold-spam). |
| **Observability** | Every state transition of a seat (AVAILABLE → HELD → BOOKED/RELEASED) and every payment state transition must be traceable end-to-end via correlation ID. | Concurrency bugs and payment discrepancies are otherwise nearly undebuggable in production. |

## Explicit Non-Goals (v1)

- Dynamic/surge pricing engine
- Seat recommendation / personalization
- Multi-currency, multi-region active-active deployment
- Loyalty programs, coupons/wallets (mentioned only as future evolution)

## Consistency Boundary (stated up front)

> **Strong consistency** is required for: seat state (per show), booking
> creation, payment state.
> **Eventual consistency** is acceptable for: notifications, catalog search
> index, admin dashboards/analytics.

This boundary is the single most important architectural decision in the
system and is referenced throughout every later document.
