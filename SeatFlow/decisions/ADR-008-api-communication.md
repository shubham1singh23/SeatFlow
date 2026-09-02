# ADR-008: REST for Synchronous Communication (No gRPC in v1)

## Context
Booking Service needs to synchronously call Payment Service (create
payment intent), and clients call every service through the gateway.

## Problem
What protocol should synchronous inter-service and client-facing calls
use?

## Options Considered

### Option 1: REST/JSON everywhere
Simple, human-debuggable, identical tooling for client-facing and
internal calls, adequate performance at this system's actual QPS
(`02-scale-estimation.md` shows low-hundreds peak QPS, not a regime where
REST's overhead is a real bottleneck).

### Option 2: gRPC for internal service-to-service calls
Lower latency/overhead, strongly-typed contracts via protobuf — genuinely
better at very high internal call volume.

### Option 3: Mixed (REST external, gRPC internal)
Best of both, but doubles the tooling/observability/debugging surface
(two serialization formats, two client stacks) for a project whose actual
internal call volume doesn't demand it.

## Decision
REST/JSON for both external and internal synchronous calls in v1.

## Why We Chose It
The one synchronous internal call on the critical path (Booking →
Payment, intent creation) is low-volume relative to what would make
gRPC's overhead reduction actually matter (see scale estimation). Using
one protocol everywhere keeps observability, debugging, and onboarding
simpler — optimizing for engineering clarity over a performance gain that
isn't needed at this scale.

## Trade-offs
- Marginally higher serialization overhead than gRPC for internal calls —
  immaterial at the estimated QPS.
- No compile-time contract enforcement between services (mitigated with
  OpenAPI schemas and contract tests, not built here but noted).

## Consequences
- One HTTP client stack, one observability approach (traces/metrics) for
  all synchronous calls.

## Failure Scenarios
N/A — this is a protocol choice, not a failure-mode-bearing decision on
its own; failure handling for the Booking→Payment call is covered in
`11-reliability-and-failure-handling.md` (circuit breakers/timeouts).

## Future Evolution
If internal call volume between Booking and Payment (or any future
internal call) grows to where serialization/connection overhead is a
measured bottleneck, gRPC is the natural next step for that specific
path — not a wholesale rewrite.
