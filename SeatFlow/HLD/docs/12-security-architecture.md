# 12 — Security Architecture

## Authentication

- Password storage: bcrypt (adaptive cost), never reversible encryption.
- Login issues a short-lived **JWT access token** (e.g. 15 min) and a
  longer-lived **refresh token** (e.g. 7–30 days), refresh token rotated
  on use and stored hashed server-side so it can be revoked.
- API Gateway verifies JWT signature/expiry on every request and forwards
  identity as signed internal headers (`X-User-Id`, `X-User-Roles`) —
  downstream services trust the gateway boundary and don't re-parse JWTs,
  keeping token-verification logic in one place.

## Authorization (RBAC)

| Role | Capabilities |
|---|---|
| `ROLE_USER` | Browse, hold, book, cancel own bookings, view own history |
| `ROLE_VENUE_MANAGER` | All of the above + manage shows/pricing for their own venue(s) |
| `ROLE_ADMIN` | Full catalog management across all venues |

Enforced per-endpoint at the service layer (not just gateway routing),
because a compromised/misrouted internal call must still fail authorization
correctly — defense in depth, not a single perimeter check.

## Input Validation

Two layers, deliberately not just one:
1. **Schema/type validation** at the controller boundary (Bean Validation)
   — rejects malformed requests cheaply, before touching business logic.
2. **Domain invariant validation** inside the transactional service layer
   (e.g. "seat must currently be AVAILABLE") — never trusts that a
   request which passed schema validation is *semantically* safe to act
   on; this is also what the concurrency design in `07` depends on.

## Rate Limiting

Redis-backed fixed-window counters, applied at the gateway/service edge:

| Endpoint class | Limit rationale |
|---|---|
| `POST /bookings/holds` | Prevent hold-spam bots from artificially locking out real users on hot shows |
| `POST /auth/login` | Brute-force protection |
| General API | Coarse per-user/IP ceiling to protect shared infrastructure |

## Webhook Security

Payment provider webhooks are verified via **HMAC signature** using a
provider-issued shared secret before any processing occurs; unsigned or
invalid-signature requests are rejected with no state change. Combined
with `eventId` idempotency (`10-payment-architecture.md`), this prevents
both forged and duplicated webhook processing.

## Service-to-Service Communication

v1: internal REST calls within a private network/VPC, not exposed
publicly; the gateway is the only public ingress. Internal identity
headers are only trusted because they originate from the gateway inside
the trusted network boundary.

## Secrets Management

Database credentials, JWT signing keys, and the payment-provider webhook
secret are injected via environment/secret-store (e.g. Kubernetes
Secrets), never committed to source control.

## Implemented vs. Production-Hardening Recommendation

| Concern | Status |
|---|---|
| JWT auth, RBAC, bcrypt hashing, webhook HMAC verification, basic rate limiting, input validation | **Implemented** in this design |
| mTLS between internal services | **Production hardening recommendation** — not required at this scale/threat model for a portfolio project, but the natural next step for a real deployment on shared infrastructure |
| WAF / DDoS protection at the edge | **Production hardening recommendation** — typically provided by the cloud load balancer / CDN layer, out of scope for application-level design |
| Secrets rotation automation, full audit logging pipeline | **Production hardening recommendation / future implementation** |
| Field-level encryption for PII at rest | **Production hardening recommendation**, depending on regulatory requirements not specified in v1 scope |
