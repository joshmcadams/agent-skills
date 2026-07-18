# Repositories

A **repository** gives the domain the illusion of an in-memory collection of
aggregates: get one by Id, get them all, add, update, remove. Behind the illusion
sits a database the domain never sees, because of one structural decision:

**The interface lives in the domain. The implementation lives in infrastructure.**

## The dependency arrow points inward

The interface, as the domain knows it, names no database type:

```go
package repositories

type ProductRepository interface {
	Create(ctx context.Context, product *entities.ValidatedProduct) (*entities.Product, error)
	FindById(ctx context.Context, id uuid.UUID) (*entities.Product, error)
	FindAll(ctx context.Context) ([]*entities.Product, error)
	Update(ctx context.Context, product *entities.ValidatedProduct) (*entities.Product, error)
	Delete(ctx context.Context, id uuid.UUID) error
}
```

Notice what's absent — no `*sql.DB`, no pgx types, no query strings — and what's
present: `ValidatedProduct` in the write signatures, the type-level guarantee
from [`entities.md`](entities.md) that nothing unvalidated reaches storage.

Infrastructure depends on the domain by *implementing* an interface the domain
owns. This dependency inversion does real work: application services are testable
with an in-memory fake (milliseconds, no Docker), the storage engine is swappable
(the reference project moved GORM → sqlc without touching domain or application),
and every capability the domain grants persistence is enumerated in one small
interface — no "just run a quick query" from a service.

## One repository per aggregate root

There is a `ProductRepository` and a `SellerRepository`, and there will never be
a `ProductEventRepository` or a `MoneyRepository`. You load and store
*aggregates*, whole, by their root ([`aggregates.md`](aggregates.md)). Nothing is
ever half-loaded; nothing inside a boundary is saved independently.

## The implementation: sqlc, not an ORM

The concrete side is built on [sqlc](https://sqlc.dev) — you write real SQL, sqlc
generates type-safe Go at build time. In a DDD codebase the row-to-aggregate
mapping is a *boundary where invariants leak*, and you want it explicit and dumb:
no reflection deciding what loads when, no `save()` cascading through an object
graph the aggregate design says shouldn't exist. Nothing in the architecture
*requires* sqlc — the interface doesn't know it exists — but it fits the goal.

## Reconstruction routes through the constructors

Reading a row back is a boundary crossing, so it revalidates — same rule as the
JSON boundary in [`value-objects.md`](value-objects.md):

```go
func productFromRow(id uuid.UUID, name string, priceCents int64, currency string, /* ... */) (*entities.Product, error) {
	price, err := entities.NewMoney(priceCents, entities.Currency(currency))
	if err != nil {
		return nil, err
	}
	return &entities.Product{Id: id, Name: name, Price: price /* ... */}, nil
}
```

If someone hand-edits a row to `currency = 'XXX'`, the repository refuses to
materialize it rather than letting corrupt money into the domain. (This is the
one place a struct literal for an entity is legitimate — it's *inside* the
`entities`-adjacent reconstruction boundary and every value went through a
validating constructor.)

## `find` vs `get`, and errors as sentinels

- **`find` may return nothing** — absence is a normal outcome. `FindById` returns
  `(nil, nil)` on a missing row so the application layer can decide it's a 404:

  ```go
  if errors.Is(err, pgx.ErrNoRows) {
      return nil, nil // FindById: absence is not an error
  }
  ```

- **`get` must return a value** — absence *is* an error. And an `Update` that
  matches no row means the caller acted on something that doesn't exist:

  ```go
  if rows == 0 {
      return nil, entities.ErrProductNotFound // a domain sentinel
  }
  ```

Infrastructure errors (pgx types) **must not leak** above infrastructure; domain
sentinels do the traveling, so the interface layer maps them with `errors.Is`.

## Transactions live in the repository

The repository owns the transaction, not the service. `Create` opens one, writes
the aggregate row *and* its domain events (the outbox —
[`domain-events-outbox.md`](domain-events-outbox.md)), reads the row back, and
commits — all inside one `pgx` transaction:

```go
tx, err := repo.pool.Begin(ctx)
defer func() { _ = tx.Rollback(ctx) }()
qtx := repo.queries.WithTx(tx)
// insert aggregate row + insert outbox events + read back
return created, tx.Commit(ctx)
```

The unit of consistency is already decided by the aggregate boundary — one
aggregate, one transaction — and the repository *is* that unit, so the transaction
hides behind the interface. The day a use case genuinely needs to commit two
aggregates atomically, revisit the aggregate boundary first; don't hand every
service a transaction handle.

## Read after write

After a write, read the row back before returning it (inside the same
transaction). This confirms the data was written correctly and means callers never
operate on a stale in-memory copy. See [`conventions.md`](conventions.md).

## Soft deletes

`Delete` sets `deleted_at = NOW()`; it does not remove the row. Every read query
filters `deleted_at IS NULL`. "Deleted products don't exist" is thereby enforced
in every read path, in one reviewed place. See [`conventions.md`](conventions.md).
