# Architecture — the onion and its layers

Dependencies point **inward only**. The domain sits at the center and imports
nothing from the layers around it. This is the single rule the whole structure
serves; everything else is a consequence.

```
internal/
├── domain/           # entities, value objects, events, repository INTERFACES
├── application/      # commands, queries, services (orchestration)
├── infrastructure/   # Postgres/sqlc repositories, outbox relay, config
└── interface/        # REST controllers, DTOs
```

## The import rules

| Layer | May import | Must never import |
|---|---|---|
| `domain` | stdlib, `google/uuid` | application, infrastructure, interface, any framework or driver (Echo, pgx, sqlc, net/http, encoding for DB) |
| `application` | domain | infrastructure, interface, Echo, pgx |
| `infrastructure` | domain, application | interface |
| `interface` | application, domain (types only) | infrastructure |

The wiring point where every layer meets is `cmd/<service>/main.go`: it
constructs concrete repositories, injects them into services, and hands services
to controllers. **No other file knows all the layers** — that is how you keep the
arrows honest. If a package outside `main` imports across a forbidden boundary,
the layering has leaked.

Why it matters, concretely: HTTP frameworks and database libraries churn; "a
product belongs to a seller" does not. A domain free of infrastructure is a
domain whose business knowledge survives every migration — the reference project
moved from GORM to sqlc/pgx without touching a domain rule.

## Layer responsibilities

**Domain** — the model. Entities enforce invariants via constructors and
`validate()`; value objects like `Money` are unconstructible in invalid states;
aggregates reference each other by Id only; repository *interfaces* declare what
persistence the model needs. Errors are sentinels (`ErrValidation`,
`ErrProductNotFound`) that outer layers translate. See
[`entities.md`](entities.md), [`value-objects.md`](value-objects.md),
[`aggregates.md`](aggregates.md).

**Application** — the use cases. Commands and queries as explicit types; services
orchestrate (load aggregates, invoke domain behavior, persist) but **never decide
business rules**; a `withIdempotency` decorator makes every command retry-safe.
Results are output shapes, not entities. See [`cqrs.md`](cqrs.md),
[`idempotency.md`](idempotency.md).

**Infrastructure** — the details. sqlc-generated, type-safe SQL behind the
domain's repository interfaces; row-to-entity mapping routes through the
validating constructors; the aggregate write and its outbox events share one
transaction; the relay polls and publishes with at-least-once semantics. See
[`repositories.md`](repositories.md),
[`domain-events-outbox.md`](domain-events-outbox.md).

**Interface** — the edge. Controllers parse DTOs, call services, and map sentinel
errors to status codes. DTOs use explicit primitives (`price_cents`, not `price`)
so the wire format can never be ambiguous, and never expose domain types
directly.

## The request flows

**Write path:** controller → command → service → domain (`NewMoney`,
`NewProduct`, `NewValidatedProduct`) → repository (one transaction: aggregate row
+ outbox events + read-back) → result → response. The service reserves an
idempotency key first and orchestrates; the domain enforces every rule.

**Read path:** controller → query → service → repository → result mapping. No
entity construction, no idempotency, no events — reads have no business rules, so
they skip the write-path ceremony entirely.

**Event path:** the outbox relay polls `outbox_events WHERE published_at IS NULL`
(a partial index keeps that query cheap forever), hands each event to a
`Publisher`, and marks it published. At-least-once; consumers deduplicate on the
UUIDv7 event Id.

## DDD vs Clean vs Hexagonal

Same picture at the resolution that matters: dependencies point inward, the
domain doesn't know about infrastructure, and the boundary is interfaces the
inner layer owns. Hexagonal calls them ports and adapters; Clean Architecture
calls them use cases and gateways; DDD adds the *modelling* vocabulary —
entities, value objects, aggregates — the other two are largely silent about.
This structure is onion-layered in shape and DDD in modelling:
`internal/domain/repositories` are the ports, `internal/infrastructure/db` are
the adapters.

See [`conventions.md`](conventions.md) for the cross-cutting house rules and
[`tooling.md`](tooling.md) for the concrete stack that fills each layer.
