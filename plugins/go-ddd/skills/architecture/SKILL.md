---
name: architecture
description: The onion/layered architecture for Domain-Driven Go services — the four layers (domain, application, infrastructure, interface), the dependency rules that keep imports pointing inward, where each responsibility lives, and how a request flows. Use when starting or structuring a Go backend, deciding which layer code belongs in, reviewing whether a layer boundary was violated, or explaining how DDD/onion/hexagonal map onto a Go project.
type: knowledge
---

# Go DDD Architecture — layers and the inward dependency rule

The structural foundation every other skill in this plugin builds on: a Go
backend split into four layers whose dependencies point **inward only**, with the
domain at the center importing nothing around it. Get this right first — a leaked
import here undermines every pattern layered on top.

> **These are opinions, not a standard.** This guidance is adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd) — a coherent,
> opinionated house style, not an external specification. See the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE) for attribution and the deliberate
> choices (`Id` vs `ID`, sqlc over an ORM, one bounded context). Where a rule
> states a *why*, apply it where the why holds.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root
> is the directory two levels above this SKILL.md.)

This is a **knowledge** skill — it produces guidance, not edits to an artifact.
Advise; do not modify files unless asked. To actually add a feature across these
layers, use **add-aggregate**; to check an existing repo against the rules, use
**audit**.

## The four layers

```
internal/
├── domain/           # entities, value objects, events, repository INTERFACES
├── application/      # commands, queries, services (orchestration)
├── infrastructure/   # Postgres/sqlc repositories, outbox relay, config
└── interface/        # REST controllers, DTOs
```

## The import rules (the whole point)

| Layer | May import | Must never import |
|---|---|---|
| `domain` | stdlib, `google/uuid` | application, infrastructure, interface, any framework/driver (Echo, pgx, sqlc, net/http) |
| `application` | domain | infrastructure, interface, Echo, pgx |
| `infrastructure` | domain, application | interface |
| `interface` | application, domain (types only) | infrastructure |

Everything meets in `cmd/<service>/main.go`: it constructs concrete repositories,
injects them into services, and hands services to controllers. **No file outside
`main` knows all four layers.** If a non-`main` package imports across a forbidden
boundary, the layering has leaked — that's the first thing to flag in review.

Why it earns its keep: HTTP frameworks and DB libraries churn; business rules
("a product belongs to a seller") don't. A domain free of infrastructure is a
domain whose rules survive every migration.

## What goes in each layer

- **Domain** — the model, the layer with the fewest imports. Entities enforce
  invariants via constructors + `validate()`; value objects are unconstructible
  when invalid; aggregates reference each other by Id; repository *interfaces*
  declare what persistence the model needs; errors are sentinels outer layers
  translate.
- **Application** — the use cases. Commands/queries as explicit types; services
  orchestrate (load aggregates, invoke domain behavior, persist) but **never
  decide business rules**; a `withIdempotency` decorator makes commands
  retry-safe. Results are output shapes, not entities.
- **Infrastructure** — the details. sqlc-generated SQL behind the domain's
  interfaces; row→entity mapping routes through validating constructors; aggregate
  write and outbox events share one transaction; the relay publishes events.
- **Interface** — the edge. Controllers parse DTOs, call services, map sentinel
  errors to status codes. DTOs use explicit primitives (`price_cents`, not
  `price`) and never expose domain types.

## Deciding which layer code belongs in

Ask, in order:

1. **Is it a business rule or invariant?** → domain (in an entity/value object).
   If a second delivery mechanism (CLI, queue consumer) could get *different*
   business behavior by calling things differently, the rule is in the wrong layer.
2. **Is it orchestration** — loading aggregates, sequencing calls, idempotency,
   mapping to results — with no rule of its own? → application (a service).
3. **Does it touch a database, broker, framework, or the network?** → infrastructure.
4. **Is it parsing/serializing the wire format or routing HTTP?** → interface.

A rule of thumb for the common mistake: *business logic in an HTTP handler* is the
symptom that pushed you to this architecture in the first place — move it down to
the domain.

## How a request flows

- **Write:** controller → command → service (reserve idempotency key, orchestrate)
  → domain (`NewMoney`, `NewProduct`, `NewValidatedProduct`) → repository (one
  transaction: row + outbox events + read-back) → result → response.
- **Read:** controller → query → service → repository → result mapping. No entity
  construction, no idempotency, no events — reads have no business rules.
- **Event:** the outbox relay polls unpublished rows, publishes each, marks it
  published — at-least-once; consumers dedupe on the event Id.

## DDD vs Clean vs Hexagonal

Same picture at the resolution that matters — dependencies inward, domain unaware
of infrastructure, boundary owned by the inner layer. Hexagonal: ports & adapters
(`internal/domain/repositories` are ports, `internal/infrastructure/db` are
adapters). Clean: use cases & gateways. DDD adds the *modelling* vocabulary. This
structure is onion in shape, DDD in modelling.

## Output

When advising, state: which layer the code in question belongs in and why, any
import-rule violation you see (name the offending import and the rule it breaks),
and the concrete move to fix it. Point to the relevant deeper skill
(**entities**, **value-objects**, **aggregates**, **repositories**, **cqrs**,
**domain-events-outbox**, **idempotency**, **testing**, **conventions**) for the
details of whatever the code needs next.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/architecture.md` — the layers, import rules, request flows, and the
  DDD/Clean/Hexagonal mapping
- `reference/conventions.md` — the cross-cutting house rules the layers rely on
- `reference/tooling.md` — the concrete stack that fills each layer
- `reference/index.md` — chapter map for the deeper per-pattern references
