# 17 — Package Structure

```
com.seatflow.booking
│
├── api
│   ├── BookingController
│   ├── dto/            CreateBookingRequestDto, BookingResponseDto, ...
│   └── mapper/          BookingMapper
│
├── application
│   └── BookingApplicationService
│
├── domain
│   ├── model/            Booking, BookingItem, BookingStatus, BookingItemStatus,
│   │                       Money, HoldToken
│   ├── service/           BookingDomainService, SeatHoldDomainService, BookingFactory
│   ├── event/             BookingHeld, BookingConfirmed, BookingCancelled,
│   │                       BookingExpired, BookingFailed (payload records)
│   └── exception/          SeatUnavailableException, InvalidTransitionException,
│                            IdempotencyConflictException
│
├── infrastructure
│   ├── persistence/       BookingRepository, BookingItemRepository, JPA entities
│   ├── redis/               SeatHoldCoordinator
│   ├── outbox/               OutboxEventEntity, OutboxService, OutboxRelay
│   ├── idempotency/          IdempotencyRecordEntity, IdempotencyService
│   └── clients/                PaymentServiceClient, ShowServiceClient (+ their DTOs)
│
└── configuration
    ├── TransactionConfig, RedisConfig, KafkaProducerConfig, WebClientConfig
```

## Reasoning
Layering follows dependency direction strictly inward:
`api → application → domain ← infrastructure`. `domain` has **zero**
dependency on `infrastructure` or `api` — `infrastructure` implements
interfaces `domain` defines (e.g. `SeatHoldCoordinator` interface lives
conceptually with domain's needs even though its Redis implementation
lives in `infrastructure/redis`; for a service this size we keep the
interface + impl co-located in `infrastructure/redis` rather than
introducing a `domain/port` package purely for ceremony — a explicit,
documented simplification, not an oversight). This keeps
`domain/service/BookingDomainService` and `SeatHoldDomainService`
testable with zero Spring context, satisfying Dependency Inversion without
over-abstracting for a bounded-context service of this size.
