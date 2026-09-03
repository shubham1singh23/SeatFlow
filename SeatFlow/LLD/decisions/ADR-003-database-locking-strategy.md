# ADR-003: Database Locking Strategy — Constraint-Based, Not Lock-Based

## Context
Given ADR-001 selects Postgres as the correctness authority, the question
of *how* Postgres enforces it (locks vs. constraints vs. isolation level)
still needs an explicit answer.

## Problem
`SELECT ... FOR UPDATE` and `SERIALIZABLE` isolation are the "textbook"
answers to concurrent booking systems. Are they necessary here?

## Options considered
- Pessimistic row lock on a seat-claim row held for the transaction.
- `SERIALIZABLE` isolation with retry-on-serialization-failure.
- `READ COMMITTED` + a partial unique index, relying on the constraint
  violation itself as the conflict signal.

## Decision
`READ COMMITTED` + partial unique index (`uq_active_seat_claim` in
`docs/lld/07-database-design.md`).

## Why we chose it
A unique constraint gives the exact same guarantee `SERIALIZABLE` would
here (no two active claims on the same seat) without paying
`SERIALIZABLE`'s higher abort/retry rate on unrelated transactions, and
without holding a lock for the duration of a transaction that also talks
to Redis. The conflict is detected at the one point it can actually occur
— the `INSERT` — with no lock held before or after.

## Trade-offs
This pattern relies on the invariant being expressible as a single-column
combination unique constraint. It would not generalize as cleanly to a
more complex invariant (e.g. "at most 4 seats per booking AND at most one
claim per seat" — the latter is still constraint-friendly; a genuinely
cross-row aggregate invariant might force a different mechanism).

## Consequences
Every seat-claim attempt is an `INSERT`, never an `UPDATE` of a
pre-existing "available" row — this shapes the whole `BookingItem`
lifecycle (rows are only ever created once per active claim, never
reused/recycled by resetting a status back to available).

## Failure scenarios
Constraint violation surfaces as `SQLState 23505`, mapped in
`BookingRepository` to `SeatUnavailableException` — never leaked as a raw
DB exception to the API layer.

## Future evolution
None anticipated at current scale; would revisit only if invariant
complexity grows beyond what a unique index can express.
