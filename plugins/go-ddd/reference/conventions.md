# Conventions — the cross-cutting house rules

The rules that span layers and keep the codebase consistent. Most are enforced by
review, some by the compiler, one by a deliberately-disabled linter. See
[`NOTICE`](../NOTICE) — several of these are opinions, chosen on purpose.

## Ubiquitous language

One vocabulary, shared by developers and domain experts, used *everywhere* — in
conversation, tickets, and code. If the business says "seller," the struct is
`Seller`, the table is `sellers`, the endpoint is `/sellers`. No `Vendor` in the
API and `Merchant` in the database because two developers had different tastes.
Every business question ("can a product exist without a seller?") gets answered in
exactly one place in the code.

## Constructors everywhere

`NewX` for every entity and value object. A struct literal for a domain type
**outside the `entities` package and its tests** is a review flag — it bypasses
the constructor and its validation. The one legitimate exception is row
reconstruction inside the repository, where every field has already passed a
validating constructor ([`repositories.md`](repositories.md)).

## `ValidatedX` types at trust boundaries

Repository write methods accept only `ValidatedX` types, so the compiler enforces
"this went through validation" at the persistence boundary
([`entities.md`](entities.md)). Read methods return the **plain** entity type, not
`ValidatedX` — see below.

## Don't return `ValidatedX` from reads

Read methods return the domain entity directly, never `ValidatedX`. Validation
logic changes over time; if reads demanded today's validation, you couldn't load
historical rows written under yesterday's rules without runtime errors, and you'd
be forced to migrate all your data. Guarantee you can always load historical data;
push validation to the **write** side (`NewX`, update methods) where you must
enforce invariants anyway.

## Sentinel errors, wrapped with `%w`

Domain errors are sentinels (`ErrValidation`, `ErrProductNotFound`,
`ErrSellerNotFound`), wrapped at the raise site with `fmt.Errorf("%w: detail",
ErrX)` and matched with `errors.Is`. **Never branch on error strings.**
Infrastructure error types (pgx, etc.) must not leak above infrastructure; the
interface layer maps domain sentinels to status codes (e.g. `ErrProductNotFound` →
404, `ErrValidation` → 400, `ErrRequestInFlight` → 409, `ErrIdempotencyKeyReuse` →
422).

## `find` vs `get`

- **`find`** may return nothing: `(nil, nil)` or an empty slice is a valid result.
- **`get`** must return a value; absence is an error (a domain sentinel).

## Defaults live where the invariant lives

Business defaults — the Id (a UUIDv7), `created_at`, other computed initial state
— are set in the **domain** constructor, never in the infrastructure layer and
never in the database (no `DEFAULT now()`, no DB-generated ids). Two sources of
truth for a default is dangerous; it's easier to test in one place; and databases
get replaced. One source of truth: the factory.

## Read after write

When writing to storage, read the written row back before returning it (inside the
same transaction). Confirms the write succeeded and means you never operate on a
stale in-memory copy.

## Soft deletes

Always soft-delete: a `deleted_at TIMESTAMPTZ` column set to `NOW()` on delete,
never a physical `DELETE`. Every read query filters `deleted_at IS NULL`. Deleted
data is invisible, not gone — you can restore it, and audits can still see it.

## `Id`, not `ID`

House style, enforced on purpose — the staticcheck rules that would flag it are
disabled deliberately in `.golangci.yml`. `SellerId` composes better than
`SellerID` in a codebase full of generated code and JSON tags. Consistency beats
the style guide's preference; pick one and apply it everywhere. (This is a
defensible choice, not a universal rule — flip it if you prefer, but flip it
everywhere.)

## Migrations are append-only

Schema evolves through numbered `migrations/` pairs (`NNNNNN_name.up.sql` /
`.down.sql`). Never modify a migration already applied in production; add a new
one. Always write both `up` and `down`. Keep them small and focused. sqlc
regenerates the query layer from `sql/queries/` via `make sqlc`.
