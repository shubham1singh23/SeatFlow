# ADR-001: Microservices (Small Set) Over Monolith / Modular Monolith

## Context
SeatFlow has one component with a hard correctness requirement (Booking)
and several components with very different availability/scaling profiles
(Catalog is read-heavy and must stay up; Notification is best-effort and
must never block anything).

## Problem
How should the system be decomposed so that a failure or load spike in a
low-stakes component (Notification) can never affect a high-stakes
component (Booking), and each can scale to its own actual demand shape?

## Options Considered

### Option 1: Monolith
Single deployable, single DB. Simplest to build and operate.

### Option 2: Modular Monolith
Single deployable, internally modularized by domain, one DB (or schemas).

### Option 3: Microservices (small, domain-aligned set)
Independently deployable services, each owning its own data.

## Decision
Microservices, limited to **5 services** aligned to bounded contexts:
Identity, Catalog, Booking, Payment, Notification.

## Why We Chose It
- Booking's correctness/consistency needs are fundamentally different
  from Catalog's availability needs and Notification's fire-and-forget
  needs — coupling their deployment and failure domain is the wrong
  default once that difference is identified, not before.
- Independent scaling: Catalog (read-heavy, cache-friendly) and Booking
  (write-sensitive, contention-bound) have opposite scaling profiles.
- Failure isolation: a Notification Service outage must be structurally
  incapable of affecting booking — this is only true if it's a separate
  deployable with no synchronous dependency from Booking.

## Trade-offs
- More operational surface: 5 databases, inter-service network calls,
  distributed tracing needed to debug a single user flow.
- Some genuine coupling costs: Booking Service does need Catalog data
  (show/seat existence, price) and must call it or snapshot it rather
  than joining a table directly.

## Consequences
- Cross-service data access must go through APIs or events, never shared
  DB access (see `04-service-boundaries.md`).
- Requires the Outbox Pattern (ADR-007) to avoid dual-write bugs at
  service boundaries.

## Failure Scenarios
- A crashed Notification instance cannot cascade to Booking — no
  synchronous call exists between them.
- A Catalog outage degrades browsing but does not block confirming an
  already-held booking (Booking snapshots what it needs at hold time).

## Future Evolution
At meaningfully smaller scale (single city, single chain), a modular
monolith would be the more honestly appropriate choice — this decision is
scale- and requirement-driven, not a universal default (see
`15-hld-tradeoffs.md`).
