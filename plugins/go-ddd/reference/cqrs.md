# CQRS — commands and queries

CQRS here is the modest, recommended version: **separate code paths for writes
and reads, one database.** No command bus, no separate read store, no event
sourcing. The idea in one sentence: a request either *changes* the system or
*asks* it something, and pretending those are the same operation makes both worse.

## Why split them

Writes and reads want different things:

- A **write** ("create this product") cares about invariants, validation,
  idempotency, events. It flows through the full domain model, because that's
  where the rules live.
- A **read** ("list all products") cares about shape and speed. It has *no
  business rules* — running a list query through entity construction and
  validation adds cost and implies rules that aren't there.

One `Save()`-style method serving both gives you the classic entanglement: read
endpoints paying write-path costs, and write logic leaking into whatever shape the
UI wanted this sprint.

## Commands

A command is a plain struct naming an intent in the ubiquitous language, carrying
raw-ish data (`PriceCents int64`, not `Money`) — turning that into validated
domain objects is the service's job:

```go
type CreateProductCommand struct {
	IdempotencyKey string
	Id             uuid.UUID
	Name           string
	PriceCents     int64
	Currency       entities.Currency
	SellerId       uuid.UUID
}

type CreateProductCommandResult struct {
	Result *common.ProductResult
}
```

The service *orchestrates*; it doesn't decide:

```go
func (s *ProductService) CreateProduct(ctx context.Context, cmd *command.CreateProductCommand) (*command.CreateProductCommandResult, error) {
	return withIdempotency(ctx, s.idempotencyRepo, cmd.IdempotencyKey, cmd, func() (*command.CreateProductCommandResult, error) {
		validatedSeller, err := s.findValidatedSeller(ctx, cmd.SellerId)
		if err != nil {
			return nil, err
		}
		price, err := entities.NewMoney(cmd.PriceCents, cmd.Currency)
		if err != nil {
			return nil, err
		}
		newProduct := entities.NewProduct(cmd.Name, price, *validatedSeller)
		validatedProduct, err := entities.NewValidatedProduct(newProduct)
		if err != nil {
			return nil, err
		}
		if _, err := s.productRepository.Create(ctx, validatedProduct); err != nil {
			return nil, err
		}
		return &command.CreateProductCommandResult{
			Result: mapper.NewProductResultFromValidatedEntity(validatedProduct),
		}, nil
	})
}
```

Read it as a checklist of the other patterns doing their jobs: idempotency wrapper
([`idempotency.md`](idempotency.md)) makes it retry-safe; the seller is loaded and
*validated* before use; raw ints become `Money` through the constructor; the
entity is created and validated ([`entities.md`](entities.md)); the repository
demands the validated type and persists aggregate + events atomically. Every
business rule the service *appears* to enforce is actually enforced by a domain
type it merely invokes. The test for correct layering: could a second delivery
mechanism (a CLI, a consumer) get *different* business behavior by calling things
differently? Here, no — the domain won't let it.

## Queries

The read side is deliberately thinner. A query names a question; its result is a
dumb output shape — **not the entity**:

```go
type GetProductByIdQuery struct {
	Id uuid.UUID
}

type ProductResult struct {
	Id        uuid.UUID
	Name      string
	Price     entities.Money
	SellerId  uuid.UUID
	CreatedAt time.Time
	UpdatedAt time.Time
}
```

`ProductResult` looks like the entity today but the distinction is load-bearing:
results are **outputs**, free to grow display-oriented fields (a joined-in seller
name, a computed label) without touching the domain model, and consumers of
results can't call `UpdatePrice` on one. The entity's methods and events stay
inside the write path. This split is also your performance escape hatch — a hot
list endpoint can drop to hand-tuned SQL or a denormalized view *without any
write-path change*.

## Command results, not bare entities

Commands return results (`CreateProductCommandResult` wrapping a `ProductResult`)
rather than entities. Pragmatically, HTTP clients need the created resource back
(the Id, timestamps) without a second round trip. Architecturally, returning a
result keeps the controller unable to touch domain behavior — the entity, with
its `PullEvents` and mutation methods, never crosses into the interface layer.

## What this deliberately doesn't do

- **No command/query bus** — services are called directly through interfaces. A
  bus pays off with cross-cutting middleware at scale; a codebase should show the
  pattern, not the framework.
- **No separate read store** — one Postgres. The split is code-level, where ~90%
  of the benefit lives.
- **No event sourcing** — domain events here are *notifications*
  ([`domain-events-outbox.md`](domain-events-outbox.md)), not the source of truth.

If the system grows into needing those, the seams are already in the right places.
That's the bet: not that you'll need the heavy machinery, but that adding it later
shouldn't require moving walls.
