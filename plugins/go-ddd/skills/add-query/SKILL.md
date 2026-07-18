---
name: add-query
description: Add a new read use case (query) to a Go DDD service — a query struct naming the question, a dumb result output shape (not the entity), a thin service method with no business rules, and the read repository method + sqlc query behind it. Use when the user asks to add a read/list/lookup endpoint or a filtered query (e.g. "list products cheaper than X", "get orders by seller") to an aggregate that already exists.
type: builder
argument-hint: "<question>  e.g. ProductsBySeller"
---

# Add a Query (read use case)

Add one read use case. The read side is deliberately thin: no idempotency, no
events, no entity construction ceremony — reads have no business rules.

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This skill **modifies the codebase**. Produce the new/edited files and a summary.

## Arguments

`$ARGUMENTS` names the question (`ProductsBySeller`, `GetOrderById`). The aggregate
must already exist.

## Step 0 — Orient

Read an existing query end-to-end: the struct + result in
`internal/application/query/`, the service method, the result mapper, and the read
repository method + sqlc query it uses. Mirror the idioms.

## Step 1 — Query + result

In `internal/application/query/`:

```go
type ProductsBySellerQuery struct { SellerId uuid.UUID }

// A result is an OUTPUT shape, not the entity — free to add display fields
// (a joined-in seller_name) without touching the domain model.
type ProductsBySellerQueryResult struct { Result []*common.ProductResult }
```

Never return the entity type from a query — consumers must not be able to call
mutation methods, and results are free to diverge from the domain shape.

## Step 2 — Read repository method + SQL

If the access pattern doesn't exist, add it: a method on the repository interface
returning **plain entities** (`[]*entities.Product`, not `ValidatedX`), a
`-- name: ... :many|:one` sqlc query that **filters `deleted_at IS NULL`**, `make
sqlc`, and the implementation mapping rows through the reconstruction helper. This is
exactly **add-repository-method** — invoke it or follow it.

## Step 3 — Service method (thin)

Load via the repository, map each entity to a result, return. No `withIdempotency`,
no validation dance:

```go
func (s *ProductService) FindProductsBySeller(ctx context.Context, q *query.ProductsBySellerQuery) (*query.ProductsBySellerQueryResult, error) {
	products, err := s.productRepository.FindBySellerId(ctx, q.SellerId)
	if err != nil { return nil, err }
	var out query.ProductsBySellerQueryResult
	for _, p := range products {
		out.Result = append(out.Result, mapper.NewProductResultFromEntity(p))
	}
	return &out, nil
}
```

For a single-item lookup, map a missing row to `(nil, nil)` so the controller
returns 404 (**repositories**, find vs get).

## Step 4 — Expose it (if it has an endpoint)

Add a controller handler and route (`GET`), parsing params, calling the service, and
mapping the result to a response DTO. Reads don't go through `writeCommandError`;
return the appropriate status directly (200, or 404 for a missing single item).

## Verify

1. `go build ./...`, `go vet ./...`, `make lint` clean.
2. Application test with a fake for the mapping/shape; if a new SQL query was added,
   an infrastructure test against real Postgres proving the `deleted_at` filter and
   the row mapping (**testing**).

## Output

The new query/result, the service method, any new read repository method + sqlc
query + implementation, endpoint wiring, and a summary of the shape returned and
what verification ran.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/cqrs.md` — queries, results as outputs, the thin read path
- `reference/repositories.md` — read methods return plain entities; find vs get; soft-delete filter
- `reference/testing.md` — where the read test belongs
