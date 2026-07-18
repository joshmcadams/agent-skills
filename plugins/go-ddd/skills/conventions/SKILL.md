---
name: conventions
description: The cross-cutting house conventions of a Go DDD service — ubiquitous language, constructors over struct literals, ValidatedX at write boundaries but never returned from reads, sentinel errors wrapped with %w and matched by errors.Is, find vs get, defaults set in the domain (not the DB), read-after-write, soft deletes with deleted_at, Id (not ID), and append-only migrations. Use when reviewing a Go DDD codebase for consistency, naming, error handling, or delete/default/migration discipline, or when setting house style for a new service.
type: knowledge
---

# Conventions — the cross-cutting house rules

The rules that span layers and keep a Go DDD codebase consistent. Some are enforced
by the compiler, most by review, one by a deliberately-disabled linter.

> **Several of these are opinions, chosen on purpose** (notably `Id` vs `ID`, sqlc).
> Adapted from [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE) for what's a deliberate choice. (If
> `${CLAUDE_PLUGIN_ROOT}` is undefined, the plugin root is two levels above this
> SKILL.md.)

This is a **knowledge** skill — advise; don't modify files unless asked. For a
whole-repo compliance pass, use **audit**.

## The rules

- **Ubiquitous language** — one term per concept across struct, table, and route.
  No `Vendor` in the API and `Merchant` in the DB. Every business question gets one
  home in the code.
- **Constructors, not struct literals** — `NewX` for every entity and value object.
  A composite literal of a domain type outside the `entities` package and its tests
  is a review flag; the one exception is row reconstruction in the repository.
- **`ValidatedX` at write boundaries** — repository writes take `ValidatedX`
  (compile-time validation guarantee); **reads return the plain entity**. Returning
  `ValidatedX` from reads breaks loading historical rows written under older
  validation rules — push validation to the write side.
- **Sentinel errors, wrapped with `%w`** — `fmt.Errorf("%w: detail", ErrX)`, matched
  with `errors.Is`. Never branch on error strings. Infrastructure error types (pgx)
  must not leak above infrastructure; the interface layer maps domain sentinels to
  status codes (`ErrProductNotFound`→404, `ErrValidation`→400,
  `ErrRequestInFlight`→409, `ErrIdempotencyKeyReuse`→422).
- **`find` vs `get`** — `find` may return nothing (`(nil, nil)` / empty slice);
  `get` must return a value, and absence is a (sentinel) error.
- **Defaults live where the invariant lives** — Id (UUIDv7), `created_at`, initial
  state are set in the **domain** constructor, never in infrastructure and never as
  a DB `DEFAULT`. One source of truth; easier to test; databases get replaced.
- **Read after write** — read the written row back (in the same transaction) before
  returning; never operate on a stale in-memory copy.
- **Soft deletes** — a `deleted_at TIMESTAMPTZ` set to `NOW()`, never a physical
  `DELETE`; every read filters `deleted_at IS NULL`. Deleted data is invisible, not
  gone.
- **`Id`, not `ID`** — house style, enforced on purpose (the staticcheck rules are
  disabled deliberately). Composes better with generated code and JSON tags.
  Consistency beats the style guide; pick one and apply it everywhere. (Defensible,
  not universal — flip it if you prefer, but flip it everywhere.)
- **Migrations are append-only** — numbered `up`/`down` pairs; never rewrite one
  applied in production; keep them small; regenerate sqlc from `sql/queries/` via
  `make sqlc`.

## Applying these rules

- **Setting house style:** adopt the set wholesale for a new service, or decide
  consciously where to differ and document it (the `NOTICE` does this for the
  deliberate choices).
- **Reviewing:** these are the grep-able backbone of the **audit** skill — struct
  literals outside `entities`, plain entities on writes, `ValidatedX` from reads,
  `err.Error() ==` string matching, DB `DEFAULT` for id/timestamps, physical
  `DELETE`, missing `deleted_at IS NULL`, `ID` vs `Id`, rewritten migrations.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/conventions.md` — the full treatment
- `reference/architecture.md` — the layer import rules these sit alongside
- `reference/repositories.md` — find/get, soft deletes, read-after-write in context
- `reference/entities.md` — constructors, `ValidatedX`, sentinel validation errors
