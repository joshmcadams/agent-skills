---
name: repositories
description: How to design repositories in a Go DDD service — the interface lives in the domain (naming no database type), the implementation lives in infrastructure (sqlc), one repository per aggregate root, writes take ValidatedX and reads return the plain entity, find vs get semantics, row reconstruction through validating constructors, sentinel errors that don't leak pgx types, transactions owned by the repository, and read-after-write. Use when defining or reviewing a repository interface or its sqlc/pgx implementation, or deciding where a query belongs.
type: knowledge
---

# Repositories

Design guidance for **repositories** — the domain's illusion of an in-memory
collection of aggregates. The structural decision the whole pattern rests on: **the
interface lives in the domain; the implementation lives in infrastructure.**

> **These are opinions, not a standard** (sqlc is one deliberate choice). Adapted
> from [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This is a **knowledge** skill — advise; don't modify files unless asked. To add a
new access pattern to an existing repository, use **add-repository-method**; to add
a whole aggregate's repository, use **add-aggregate**.

## The rules

1. **Interface in the domain, naming no DB type.** No `*sql.DB`, no pgx, no query
   strings in the interface — just domain types. Infrastructure depends on the
   domain by implementing it (dependency inversion). This makes services testable
   with in-memory fakes and the storage engine swappable.
2. **One repository per aggregate root.** `ProductRepository`,
   `SellerRepository` — never a `ProductEventRepository` or `MoneyRepository`. You
   load/store aggregates whole, by their root (**aggregates**). Nothing half-loaded.
3. **Writes take `ValidatedX`; reads return the plain entity.** `Create`/`Update`
   take `*entities.ValidatedProduct` (the compile-time validation guarantee);
   `FindById`/`FindAll` return `*entities.Product`. **Never return `ValidatedX`
   from reads** — validation logic changes over time, and reads must be able to
   load historical rows written under older rules (**conventions**).
4. **`find` vs `get`.** `find` may return nothing — `FindById` returns `(nil,
   nil)` on a missing row (`pgx.ErrNoRows`) so the caller can decide it's a 404.
   `get` must return a value; absence is a domain-sentinel error. An `Update`
   matching no row → `entities.ErrProductNotFound`.
5. **Reconstruct rows through validating constructors.** `productFromRow` routes
   money through `entities.NewMoney`, so a corrupt row is refused, not materialized.
   (This is the one place a struct literal for an entity is legitimate.)
6. **Errors: sentinels out, pgx in.** Infrastructure error types must not leak
   above infrastructure; domain sentinels do the traveling, matched with
   `errors.Is`.
7. **The repository owns the transaction.** `Create` opens one pgx transaction,
   writes the aggregate row *and* its outbox events (**domain-events-outbox**),
   reads the row back, and commits — the aggregate boundary already fixed the unit
   of consistency as one aggregate/one transaction, so the transaction hides behind
   the interface. Don't hand services transaction handles.
8. **Read after write; soft delete.** Read the written row back before returning.
   `Delete` sets `deleted_at = NOW()`; every read query filters `deleted_at IS
   NULL`.

```go
type ProductRepository interface {
	Create(ctx context.Context, product *entities.ValidatedProduct) (*entities.Product, error)
	FindById(ctx context.Context, id uuid.UUID) (*entities.Product, error)
	FindAll(ctx context.Context) ([]*entities.Product, error)
	Update(ctx context.Context, product *entities.ValidatedProduct) (*entities.Product, error)
	Delete(ctx context.Context, id uuid.UUID) error
}
```

## Why sqlc, not an ORM

The row-to-aggregate mapping is a *boundary where invariants leak*, so you want it
explicit and dumb: compile-time-checked SQL, no reflection deciding what loads
when, no `save()` cascading through a graph the aggregate design forbids. Nothing
in the architecture requires sqlc — the interface doesn't know it exists — so
swapping it touches `internal/infrastructure/` only. See **tooling**.

## Applying these rules

- **Designing:** put the interface in `internal/domain/repositories/`, keep it to
  the access patterns the domain actually needs, `ValidatedX` on writes.
- **Reviewing:** flag DB types in the domain interface; writes taking plain
  entities; reads returning `ValidatedX`; reads missing `deleted_at IS NULL`; pgx
  errors leaking upward; transactions composed in services; missing read-after-write.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/repositories.md` — the full treatment with transaction/reconstruction code
- `reference/aggregates.md` — one repository per root; the consistency unit
- `reference/domain-events-outbox.md` — the outbox insert sharing the write transaction
- `reference/conventions.md` — find/get, soft deletes, read-after-write, defaults-in-domain
- `reference/tooling.md` — the sqlc workflow
