# Research Notes

These notes summarize the engineering principles that shaped the design —
not an exhaustive literature dump. Where a specific named pattern is used,
the canonical source is noted.

## Concurrency control (Postgres)
**Sources**: PostgreSQL documentation — Concurrency Control, chapter on
Transaction Isolation (`READ COMMITTED` vs `SERIALIZABLE`), Indexes
(partial & unique indexes).
**Finding**: `READ COMMITTED` plus a **partial unique index** gives the
same "no two active claims on one seat" guarantee as `SERIALIZABLE`
would for this specific shape of conflict (a single-row insert
uniqueness violation), at lower cost — no serialization-failure retry
storm under contention, no lock held across a transaction lifetime.
**Influence**: chosen as the authoritative correctness mechanism (ADR-001,
ADR-003) instead of `SELECT ... FOR UPDATE` or `SERIALIZABLE` isolation.

## Redis for coordination vs. cache
**Sources**: Redis documentation — `SET` command (`NX`, `PX` options),
expiry semantics; the well-known "Distributed Locks with Redis"
discussion (Redlock) and its critiques regarding clock/GC assumptions.
**Finding**: `SET key value NX PX ttl` is a correct **atomic
claim-or-fail primitive**, but a *lock held across a critical section*
(Redlock-style) introduces a correctness dependency on timing that a
single-node or naive multi-node lock cannot fully close without
additional complexity (fencing tokens, careful lease math).
**Influence**: Redis is used only as a fast, fail-open **admission
filter** in front of the real Postgres constraint (ADR-001) — never as
the sole arbiter of a claim, and never as a lock held across a
transaction.

## Idempotency
**Sources**: general REST/idempotency-key practice as documented by major
payment API providers (the pattern of client-supplied idempotency keys
scoped per operation, stored server-side with request fingerprinting).
**Finding**: the concurrency-safety of idempotency itself is usually the
overlooked part — a naive "check then insert" has the same race condition
idempotency is meant to prevent.
**Influence**: `idempotency_records` uses its own composite primary key as
the race-resolution mechanism (`INSERT ... ON CONFLICT`), mirroring the
seat-claim approach — one consistent pattern used twice, not two
different mechanisms (ADR-004).

## Transactional Outbox
**Sources**: Chris Richardson, microservices.io — "Transactional Outbox"
pattern; standard treatment of the dual-write problem in distributed
systems writing.
**Finding**: outbox is the standard, minimal-machinery answer to
"atomically commit a DB change and reliably publish a message about it"
without XA/2PC; it trades exactly-once for at-least-once plus
consumer-side idempotency.
**Influence**: adopted as-is (ADR-005); explicitly does not claim
exactly-once delivery anywhere in this LLD.

## Spring Boot transaction management
**Sources**: Spring Framework documentation — `@Transactional`,
propagation behaviors (`REQUIRED`, `MANDATORY`), Spring Data JPA
repository behavior.
**Finding**: `Propagation.MANDATORY` is the right tool to make "this
method must run inside an existing transaction" an enforced contract
rather than a comment.
**Influence**: `OutboxService.record(...)` is declared with
`Propagation.MANDATORY` (`05-service-layer-design.md`) specifically so a
future caller cannot accidentally break the outbox's atomicity guarantee
by invoking it outside a transaction.

## Production booking/reservation system design references
**Sources**: general public engineering write-ups on high-demand ticket
on-sale systems (queue-and-release admission control patterns, seat-map
hot-spot behavior during on-sale bursts).
**Finding**: the dominant real-world failure mode isn't "the lock is
wrong," it's "too many doomed requests hit the database at once for the
same few popular seats."
**Influence**: this is precisely the justification for adding Redis as an
admission filter (Option G) rather than relying on Option F alone —
correctness was already solved by F; G solves the *load* problem F alone
doesn't address.

---
**Distinguishing designed vs. tested**: every finding above informed a
design decision recorded in `decisions/`. None of it constitutes a
benchmark run against this specific implementation — no performance
numbers are claimed in this package (see `docs/lld/18-lld-tradeoffs.md`).
