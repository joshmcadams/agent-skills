---
name: add-repository-method
description: Add a new access pattern to an existing repository in a Go DDD service — a method on the domain repository interface, a sqlc query, regenerated code, and the pgx implementation, keeping writes on ValidatedX, reads on the plain entity, the deleted_at IS NULL filter, row reconstruction through validating constructors, and sentinel errors. Use when the user asks to add a repository method, a new lookup/finder (FindBySellerId, FindByStatus), or a new query behind an existing aggregate's repository.
type: builder
argument-hint: "<RepositoryMethod>  e.g. FindBySellerId"
---

# Add a Repository Method

Add one access pattern to an existing aggregate's repository: interface method → SQL
→ generated code → implementation. Every capability the domain grants persistence is
enumerated in the interface, so this change is visible in review.

> **These are opinions, not a standard** (sqlc is a deliberate choice). Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This skill **modifies the codebase**. Produce the new/edited files and a summary.

## Arguments

`$ARGUMENTS` names the method (`FindBySellerId`, `FindByStatus`, `ExistsByName`).
Follow **find vs get**: `find*` may return nothing (`(nil, nil)` / empty slice);
`get*` must return a value and treat absence as a sentinel error.

## Step 0 — Orient

Read the existing repository: the interface in `internal/domain/repositories/`, the
sqlc queries in `sql/queries/<aggregate>.sql`, and the implementation in
`internal/infrastructure/db/postgres/sqlc_<aggregate>_repository.go` (note the
`xFromRow` reconstruction helper and the `timestamptz` helpers). Mirror them.

## Step 1 — Interface method (domain)

Add to the interface, naming **no database type**:

- **Reads** return the plain entity (`(*entities.Product, error)` /
  `([]*entities.Product, error)`) — never `ValidatedX`.
- **Writes** take `*entities.ValidatedX`.

## Step 2 — sqlc query

Add to `sql/queries/<aggregate>.sql` with the right annotation (`:one`, `:many`,
`:exec`, `:execrows`). **Reads must filter `deleted_at IS NULL`** (and any joined
aggregate's too). Then `make sqlc` (or `sqlc generate`) to regenerate
`internal/infrastructure/db/sqlc/`.

```sql
-- name: GetProductsBySellerId :many
SELECT p.id, p.name, p.price_cents, p.currency, p.seller_id, p.created_at, p.updated_at
FROM products p
JOIN sellers s ON p.seller_id = s.id
WHERE p.seller_id = $1 AND p.deleted_at IS NULL AND s.deleted_at IS NULL
ORDER BY p.created_at DESC;
```

## Step 3 — Implementation (infrastructure)

Implement the method: call the generated query, and **reconstruct each row through
the validating helper** (`productFromRow`, which routes money through `NewMoney`), so
a corrupt row is refused. Map `pgx.ErrNoRows` → `(nil, nil)` for a `find` one; for a
write matching no row, return the domain not-found sentinel. Don't leak pgx errors.
If it writes, run it in a transaction and read back.

## Step 4 — Update fakes

Add the method to the in-memory fake(s) used by application tests so they still
compile and exercise the new path.

## Verify

1. `make sqlc` produced the referenced symbols; `go build ./...`, `go vet ./...`,
   `make lint` clean.
2. **Infrastructure test against real Postgres** (testcontainers): correct results,
   the `deleted_at IS NULL` filter (insert a soft-deleted row and confirm it's
   excluded), and the sentinel/`(nil,nil)` behavior. Mocking the DB here is
   worthless — see **testing**.

## Output

The interface method, the sqlc query, the regenerated code, the implementation, the
fake update, and a summary: the access pattern, its find/get semantics, and the
verification run.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/repositories.md` — interface-in-domain, reconstruction, find/get, sentinels, transactions
- `reference/conventions.md` — find/get, soft-delete filter
- `reference/tooling.md` — the sqlc regeneration workflow
- `reference/testing.md` — the real-database repository test
