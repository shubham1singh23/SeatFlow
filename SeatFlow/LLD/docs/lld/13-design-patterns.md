# 13 — Design Patterns

| Pattern | Problem | Why it fits | Where used | Alternative considered | Trade-off |
|---|---|---|---|---|---|
| **Repository** | Decouple domain/application logic from JPA/SQL details | Lets `BookingDomainService` be tested with a fake, no Spring context | `BookingRepository`, `IdempotencyRecordRepository` | Raw `EntityManager` calls inline | Slightly more boilerplate interfaces for a real gain in testability |
| **Adapter** | Booking Service must not depend on Payment/Show Service wire formats | Isolates blast radius of an external API change to one class | `PaymentServiceClient`, `ShowServiceClient` | Calling external DTOs directly from domain code | One extra mapping layer, worth it for the isolation |
| **Transactional Outbox** | Dual-write problem between DB commit and Kafka publish | The only pattern that gives an atomicity guarantee across a DB write and an eventually-published message using tools we already have (Postgres) | `OutboxService` + `OutboxRelay` | Direct produce-after-commit (unsafe), 2PC/XA (operationally heavy, poor Kafka support) | Extra table + poller process, but it is the standard, well-understood answer to this exact problem |
| **Builder** | `Booking`/`BookingItem` construction has several required fields plus validation ordering | Readable, enforces required fields at compile time | `BookingFactory` (a small builder-backed factory inside `domain/service`) | Telescoping constructor | Minor boilerplate |
| **Strategy** (narrow use) | none forced | — | *not used* — no polymorphic algorithm variation exists today (e.g. only one seat-claim algorithm, one pricing rule) | — | Explicitly **not used**: see below |

## Explicitly rejected

- **State Pattern** — considered for `BookingStatus` transitions. Rejected
  because our states have **no per-state polymorphic behavior**, only
  transition-legality rules — a simple table-driven guard function in
  `BookingDomainService` (see `03-booking-lifecycle.md`) expresses this
  more plainly than 7 classes each overriding one method. State Pattern
  earns its cost when different states need genuinely different
  *behavior* for the same operation; here they only need different
  *permission*.
- **Saga orchestration** — rejected. There is no long-running,
  multi-service transaction requiring compensating actions across several
  services in lockstep. Payment coordination is a simple
  request/callback with a well-defined local state machine, not a saga.
  If SeatFlow later adds e.g. multi-service checkout (booking + loyalty
  points + bundled merchandise) atomically, a saga would become
  justified — noted as future evolution, not built preemptively.
- **CQRS / Event Sourcing** — rejected. Booking Service's read and write
  models are the same shape; there is no reporting/read-scaling need that
  a normal read-committed query can't satisfy. Event sourcing would also
  fight the DB-unique-constraint concurrency mechanism, which relies on a
  single current-state row, not a replayed event stream.
- **Specification Pattern** — rejected. Only one seat-validity rule
  ("seat belongs to this show and isn't claimed") exists; introducing a
  composable specification object for a single rule is speculative
  generality.
- **Strategy Pattern for pricing** — rejected for now: pricing is read
  verbatim from Show Service, not computed by Booking Service. If Booking
  Service later owns promotional discount logic, a `PricingStrategy`
  interface would be a natural, justified addition.
