---
name: cqrs
description: How to structure the write and read paths in a Go DDD service with CQRS — commands and queries as explicit types, application services that orchestrate but never decide business rules, results as output shapes rather than entities, and one database (no bus, no separate read store, no event sourcing). Use when designing an application service, deciding whether logic belongs in the service or the domain, whether to return an entity or a result, or how to separate write and read code paths.
type: knowledge
---

# CQRS — commands and queries

Guidance for splitting **writes from reads** at the code level. The recommended,
modest version: separate code paths, one database. No command bus, no separate read
store, no event sourcing. A request either *changes* the system or *asks* it
something, and pretending those are the same operation makes both worse.

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This is a **knowledge** skill — advise; don't modify files unless asked. To add a
write use case use **add-command**; a read use case, **add-query**.

## Commands (writes)

- A **command** is a plain struct naming an intent in the ubiquitous language,
  carrying raw primitives (`PriceCents int64`, not `Money`) plus an
  `IdempotencyKey string`. Turning primitives into validated domain objects is the
  service's job.
- The **service orchestrates, never decides.** It wraps the work in
  `withIdempotency` (**idempotency**), loads and validates referenced aggregates,
  builds value objects through constructors, builds and validates the entity, calls
  the repository, and maps to a result. Every business rule it *appears* to enforce
  is actually enforced by a domain type it invokes.
- The test for correct layering: could a second delivery mechanism (a CLI, a queue
  consumer) get *different* business behavior by calling things differently? If yes,
  a rule leaked out of the domain.

```go
func (s *ProductService) CreateProduct(ctx context.Context, cmd *command.CreateProductCommand) (*command.CreateProductCommandResult, error) {
	return withIdempotency(ctx, s.idempotencyRepo, cmd.IdempotencyKey, cmd, func() (*command.CreateProductCommandResult, error) {
		validatedSeller, err := s.findValidatedSeller(ctx, cmd.SellerId)
		if err != nil { return nil, err }
		price, err := entities.NewMoney(cmd.PriceCents, cmd.Currency)
		if err != nil { return nil, err }
		validated, err := entities.NewValidatedProduct(entities.NewProduct(cmd.Name, price, *validatedSeller))
		if err != nil { return nil, err }
		if _, err := s.productRepository.Create(ctx, validated); err != nil { return nil, err }
		return &command.CreateProductCommandResult{Result: mapper.NewProductResultFromValidatedEntity(validated)}, nil
	})
}
```

## Queries (reads)

- The read side is deliberately thinner: a **query** names a question, and its
  result is a dumb output shape — **not the entity**. No idempotency, no events, no
  entity construction; reads have no business rules.
- `ProductResult` may look like `Product` today, but the distinction is
  load-bearing: results are outputs, free to grow display fields (a joined-in
  seller name, a computed label) without touching the domain, and consumers can't
  call `UpdatePrice` on one. This split is also the performance escape hatch — a hot
  list endpoint can drop to hand-tuned SQL or a denormalized view with no write-path
  change.

## Command results, not bare entities

Commands return results (a `...CommandResult` wrapping a `...Result`), not entities.
Clients get the created resource back (Id, timestamps) without a second round trip,
and the entity — with `PullEvents` and mutation methods — never crosses into the
interface layer.

## What to deliberately skip

No command/query bus (call services directly through interfaces), no separate read
store (one Postgres), no event sourcing (domain events here are *notifications*, not
the source of truth — **domain-events-outbox**). The seams are placed so you can add
these later without moving walls; add them when measurements demand, not before.

## Applying these rules

- **Designing:** one command/query type per intent; service methods that read as a
  sequence of domain calls; result mappers producing outputs.
- **Reviewing:** flag business `if`s in services that belong in a `validate()`;
  entities returned from services to controllers; read paths running through entity
  construction/validation; writes not wrapped in idempotency.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/cqrs.md` — the full treatment
- `reference/idempotency.md` — the `withIdempotency` decorator on every command
- `reference/entities.md`, `reference/value-objects.md` — the domain types services invoke
- `reference/repositories.md` — what the service calls to persist and load
