---
name: modify-schema
description: Add, remove, rename, or retype a field on an existing aggregate in a Go DDD service, propagating the change coherently across every layer — the domain entity and its validate(), the migration, the sqlc query and regenerated code, the repository row mapping, the command/query result shapes, and the DTOs — while respecting defaults-in-domain, minor-unit money, and soft deletes. Use when the user asks to add/remove/rename/change-the-type of a field on an existing model/aggregate (e.g. "add a description to Product", "make Order.total a Money").
type: builder
argument-hint: "<Aggregate.field> <add|remove|rename|retype>"
---

# Modify an Aggregate's Schema

Change one field on an existing aggregate and thread it through every layer so
nothing is left half-updated. A field is not a database column — it's a domain
concept that surfaces in the entity, the schema, the mapping, the results, and the
wire.

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This skill **modifies the codebase**. Produce the edited files and a summary.

## Arguments

`$ARGUMENTS` names the aggregate + field and the operation, e.g.
`Product.description add`, `Order.total retype→Money`, `Seller.nickname remove`. If
the invariants for a new/retyped field aren't stated, ask.

## Step 0 — Orient and plan the blast radius

Read the aggregate across all layers first (entity, `validated_x.go`, migration(s),
`sql/queries/<agg>.sql`, the repository impl + `xFromRow`, command/query results,
result mappers, DTOs + converters, controller). List every touch point *before*
editing — a missed layer compiles but lies.

## Step 1 — Work inside-out

### Domain

- **add / retype:** add or change the entity field; update `validate()` (a new
  invariant, or the retyped value object's constraints); update the constructor and
  any mutation methods; retype through a value object where it earns it (money →
  `Money`, an email → `Email` — pair with **add-value-object**). Set defaults in the
  domain, never the DB.
- **rename:** rename the field and every reference; keep the ubiquitous language
  consistent (struct ↔ column ↔ route ↔ DTO all move together).
- **remove:** delete the field, its validation, and its references.

### Schema

Add a migration (**add-migration**) for the column change — add/drop/rename column,
or a type change (e.g. a `price DECIMAL` → `price_cents BIGINT` + `currency` pair,
with data backfill in the `up`). Append-only; write a real `down`. Update
`sql/queries/<agg>.sql` (the `RETURNING *` / `SELECT` column lists, the `INSERT`
/`UPDATE` column lists) and `make sqlc`.

### Infrastructure

Update `xFromRow` and the `Create`/`Update` param structs to carry the field, still
routing values through validating constructors. Keep the `deleted_at IS NULL` filter.

### Application & interface

Update the command(s)/query result shapes and result mappers, then the request/
response DTOs and their `ToXCommand`/response converters. DTOs use explicit
primitives (`price_cents`, `currency`), never domain types.

## Step 2 — Compatibility check (renames/removes/retypes)

A rename or remove on a **externally consumed** wire contract (a public API field, a
persisted event payload) is a breaking change. Call it out: keep the old JSON field
as an alias or stage the removal, and remember old **outbox event payloads** already
persisted must still deserialize. Purely internal fields can change freely. Migrate
data in the `up` migration rather than leaving a nullable column the domain can't
satisfy.

## Verify

1. `make sqlc` clean; `go build ./...`, `go vet ./...`, `make lint` clean — the
   compiler catches most missed layers.
2. Domain tests updated for the new/changed invariant (`assert.ErrorIs` on
   `ErrValidation`). Infrastructure test proves the column round-trips (and any
   backfill). See **testing**.
3. Migration `up`/`down` round-trip tested against a scratch DB (or state you
   couldn't run it).

## Output

Every edited file across the layers plus the migration, and a summary: the field
change, each layer touched, whether it's wire-breaking and how that's handled, and
the verification run.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/entities.md` — the field, its invariant, the constructor/mutation methods
- `reference/value-objects.md` — retyping a primitive into a proper value
- `reference/repositories.md` — row mapping, `RETURNING`/`SELECT` lists, reconstruction
- `reference/conventions.md` — defaults-in-domain, soft deletes, ubiquitous language
- `reference/domain-events-outbox.md` — old persisted event payloads must still deserialize
