---
name: add-migration
description: Add a golang-migrate schema migration to a Go DDD service — a numbered up/down pair under migrations/, following append-only discipline, soft-delete columns, no DB-side defaults for identity/timestamps, integer minor-unit money columns, then regenerating sqlc. Use when the user asks to add or change database schema — a new table, column, index, or constraint — or to create a migration. For editing an aggregate's Go schema/fields end-to-end, pair with modify-schema.
type: builder
argument-hint: "<snake_case_change>  e.g. add_reviews_table"
---

# Add a Migration

Add one golang-migrate migration as an `up`/`down` pair, respecting append-only
discipline and the schema conventions, then regenerate sqlc.

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This skill **modifies the codebase**. Produce the new files and a summary.

## Arguments

`$ARGUMENTS` is a short snake_case description (`add_reviews_table`,
`add_products_archived_at`). Keep migrations small and focused — one logical change.

## Step 0 — Orient

List `migrations/` to find the current highest number and the naming style
(`NNNNNN_name.up.sql` / `.down.sql`). Read a couple to match conventions. Confirm the
project uses golang-migrate (a `migrate.go` and/or a `make migrate-up` target).

## Step 1 — Create the pair

Next sequential number (zero-padded to match), both directions:

- `migrations/NNNNNN_<change>.up.sql`
- `migrations/NNNNNN_<change>.down.sql`

**Never modify a migration already applied in production** — always add a new one.

Prefer the CLI if available so numbering/format is automatic:
`migrate create -ext sql -dir migrations -seq <change>`.

## Step 2 — Write the SQL to the conventions

- **Both directions must be real.** The `down` reverses the `up` (drop what you
  created, restore what you dropped). Test the round-trip.
- **Soft deletes:** new tables get a `deleted_at TIMESTAMPTZ` column (nullable) and
  an index on it; deletes are `UPDATE ... SET deleted_at = NOW()`, never `DELETE`.
- **No DB-side defaults for identity or timestamps** — no `DEFAULT
  gen_random_uuid()`, no `DEFAULT now()` on `id`/`created_at`. Those defaults live in
  the domain constructor (**conventions**). Business defaults belong in code.
- **Money is integer minor units:** `*_cents BIGINT NOT NULL` + `currency TEXT NOT
  NULL`, never a float/decimal price.
- Add indexes the read queries need (foreign-key columns, partial indexes like the
  outbox's `WHERE published_at IS NULL`).

```sql
-- NNNNNN_add_reviews_table.up.sql
CREATE TABLE reviews (
    id UUID PRIMARY KEY,
    product_id UUID NOT NULL REFERENCES products(id),
    rating SMALLINT NOT NULL,
    body TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    deleted_at TIMESTAMPTZ
);
CREATE INDEX idx_reviews_product_id ON reviews(product_id);
CREATE INDEX idx_reviews_deleted_at ON reviews(deleted_at);
```
```sql
-- NNNNNN_add_reviews_table.down.sql
DROP TABLE reviews;
```

## Step 3 — Regenerate sqlc

sqlc reads `migrations/` as its schema. After changing schema (and any
`sql/queries/`), run `make sqlc` (or `sqlc generate`) and confirm the generated
models in `internal/infrastructure/db/sqlc/` reflect the change.

## Verify

1. Migration applies and rolls back cleanly against a scratch database:
   `make migrate-up` then a `down 1` (or `go run migrate.go -command up` / `-command
   down -steps 1`). Confirm `down` fully reverses `up`.
2. `make sqlc` succeeds and downstream code still `go build ./...`s.
3. State whether you could run the migration (needs a database) or only wrote it.

## Output

The `up`/`down` pair, any regenerated sqlc, and a summary: the schema change, the
conventions honored (soft-delete column, no DB defaults, minor-unit money), and
whether the round-trip was tested.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/conventions.md` — append-only migrations, soft deletes, defaults-in-domain
- `reference/tooling.md` — golang-migrate + the sqlc workflow
- `reference/repositories.md` — how the generated code is consumed
- `reference/value-objects.md` — why money columns are integer minor units
