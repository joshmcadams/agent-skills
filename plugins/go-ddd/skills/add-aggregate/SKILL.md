---
name: add-aggregate
description: Add a new aggregate (a domain entity with its own repository, persistence, use cases, and HTTP endpoints) to a Domain-Driven Go service, wiring all four layers in the order that keeps the compiler helping you. Use when the user asks to add a new aggregate, entity, resource, or model end-to-end to a Go DDD/onion codebase (e.g. "add a Review aggregate", "add an Order model with CRUD").
type: builder
argument-hint: "<AggregateName>"
---

# Add an Aggregate

Add one new aggregate to a Domain-Driven Go service — the domain type and its
invariants, a repository interface and its sqlc implementation, the schema, the
commands/queries/service, and the REST controller — so it complies with the
architecture and conventions bundled with this plugin.

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd). See the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE) for attribution and the deliberate
> choices this style makes. (If `${CLAUDE_PLUGIN_ROOT}` is undefined, the plugin
> root is two levels above this SKILL.md.)

This skill **modifies the codebase**. Produce the new/edited files and a short
summary of every change.

## Arguments

`$ARGUMENTS` is the aggregate name in the ubiquitous language, singular, PascalCase
(e.g. `Review`, `Order`, `ShipmentOrder`). Derive the other forms from it and use
them consistently ([`conventions.md`](${CLAUDE_PLUGIN_ROOT}/reference/conventions.md)):

- Go type: `Review`; validated type: `ValidatedReview`
- package/file stems: `review` (`review.go`, `validated_review.go`, …)
- table + collection: plural snake_case `reviews`; route base `/api/v1/reviews`

If the user hasn't said what fields/invariants the aggregate has, ask before
generating — the invariants *are* the design. Don't invent business rules.

## Step 0 — Orient in the actual codebase

Before writing anything, read an existing aggregate end-to-end and **mirror its
idioms exactly** (import paths, error handling, naming, file layout). Find the
module path from `go.mod`; find the layer dirs (typically `internal/domain/`,
`internal/application/`, `internal/infrastructure/`, `internal/interface/`).

If the repo is **not** structured this way (no layer split, no existing
aggregate to copy), say so and confirm the target structure with the user before
scaffolding — don't silently impose the layout on a codebase that rejected it.
Decide, and state, whether this new type is really a **separate aggregate** or a
value object / a field on an existing aggregate: only cluster things that must be
consistent in one transaction, and reference other aggregates **by Id**
([`aggregates.md`](${CLAUDE_PLUGIN_ROOT}/reference/aggregates.md)). If it isn't its
own aggregate, stop and use the narrower skill instead (a value object → the
domain patterns in **value-objects**; a new field → **modify-schema**).

## Build order (compiler-helping, inside-out)

Work strictly domain → schema → infrastructure → application → interface, testing
each layer as you go. Each step compiles against the one before it.

### 1. Domain

`internal/domain/entities/review.go` — the entity, its `validate()`, and a `NewX`
constructor that assigns identity and defaults in the domain (never the DB):

```go
func NewReview(/* value objects + ValidatedX refs to other aggregates */) *Review {
	review := &Review{
		Id:        uuid.Must(uuid.NewV7()),
		CreatedAt: time.Now(),
		UpdatedAt: time.Now(),
		// fields...
	}
	// review.recordEvent(events.NewReviewCreated(...))  // only if other systems care
	return review
}

func (r *Review) validate() error {
	// one guard per invariant, each wrapping ErrValidation with %w
	if /* invariant violated */ {
		return fmt.Errorf("%w: <what and why>", ErrValidation)
	}
	return nil
}
```

- Constructors take **`ValidatedX`** for references to other aggregates, and
  store just their `Id` — a `Review` holds `ProductId uuid.UUID`, not a `Product`.
- Mutations go through methods ending in `return r.validate()`.
- Add the `ValidatedReview` wrapper (`internal/domain/entities/validated_review.go`):

```go
type ValidatedReview struct {
	Review
	isValidated bool
}

func NewValidatedReview(review *Review) (*ValidatedReview, error) {
	if err := review.validate(); err != nil {
		return nil, err
	}
	return &ValidatedReview{Review: *review, isValidated: true}, nil
}
```

- Add any new sentinel (`ErrReviewNotFound`) to `internal/domain/entities/errors.go`.
- The repository **interface** in `internal/domain/repositories/review_repository.go`
  — writes take `*entities.ValidatedReview`, reads return `*entities.Review`:

```go
type ReviewRepository interface {
	Create(ctx context.Context, review *entities.ValidatedReview) (*entities.Review, error)
	FindById(ctx context.Context, id uuid.UUID) (*entities.Review, error)
	FindAll(ctx context.Context) ([]*entities.Review, error)
	Update(ctx context.Context, review *entities.ValidatedReview) (*entities.Review, error)
	Delete(ctx context.Context, id uuid.UUID) error
}
```

Write **domain tests now** (pure Go, microseconds): every invariant, asserting
`assert.ErrorIs(t, err, entities.ErrValidation)` — see
[`testing.md`](${CLAUDE_PLUGIN_ROOT}/reference/testing.md). Domain tests before any
plumbing exists — they pin the rules and they're the cheapest.

### 2. Schema

- A migration **pair** in `migrations/` — next sequential number,
  `NNNNNN_add_reviews.up.sql` / `.down.sql`. Include a `deleted_at TIMESTAMPTZ`
  column (soft delete) and index it; do **not** put DB-side defaults for `id` or
  timestamps (defaults live in the domain). Money columns are `*_cents BIGINT` +
  `currency TEXT`, never a float.
- Queries in `sql/queries/reviews.sql` with sqlc annotations. Every read filters
  `deleted_at IS NULL`; `Delete` is a soft delete; `Update` is `:execrows` so the
  repo can detect "no row matched":

```sql
-- name: CreateReview :one
INSERT INTO reviews (id, /* cols */, created_at, updated_at) VALUES ($1, /* ... */) RETURNING *;
-- name: GetReviewById :one
SELECT /* cols */ FROM reviews WHERE id = $1 AND deleted_at IS NULL;
-- name: GetAllReviews :many
SELECT /* cols */ FROM reviews WHERE deleted_at IS NULL ORDER BY created_at DESC;
-- name: UpdateReview :execrows
UPDATE reviews SET /* cols */, updated_at = $N WHERE id = $1 AND deleted_at IS NULL;
-- name: DeleteReview :exec
UPDATE reviews SET deleted_at = NOW() WHERE id = $1;
```

- Run `make sqlc` (or `sqlc generate`) to regenerate `internal/infrastructure/db/sqlc/`.

### 3. Infrastructure

`internal/infrastructure/db/postgres/sqlc_review_repository.go` — implement the
interface. Mirror the existing repository exactly:

- `NewSqlcReviewRepository(pool)` returns `repositories.ReviewRepository`.
- `Create` runs in **one transaction**: insert the row, insert any outbox events
  (`insertOutboxEvents(ctx, qtx, review.PullEvents())`), read the row back, then
  `tx.Commit`.
- Reconstruct rows through the **validating constructors** in a `reviewFromRow`
  helper (route money through `entities.NewMoney`), so a corrupt row is refused,
  not materialized.
- `FindById` maps `pgx.ErrNoRows` → `(nil, nil)`; `Update` maps `rows == 0` →
  `entities.ErrReviewNotFound`. Never let pgx errors leak upward.

Write **infrastructure tests** against real Postgres via testcontainers —
soft-delete filtering, the not-found sentinel, and (if events) outbox atomicity.

### 4. Application

- Commands/results in `internal/application/command/` (create/update/delete) and
  queries/results in `internal/application/query/` (by-id, all). Commands carry
  raw primitives (`PriceCents int64`), not domain value objects, plus an
  `IdempotencyKey string`.
- A `ReviewService` in `internal/application/services/` implementing an interface
  in `internal/application/interfaces/`. **Wrap every write in `withIdempotency`**;
  inside, load referenced aggregates and validate them, build value objects
  through constructors, build + validate the entity, then call the repo:

```go
func (s *ReviewService) CreateReview(ctx context.Context, cmd *command.CreateReviewCommand) (*command.CreateReviewCommandResult, error) {
	return withIdempotency(ctx, s.idempotencyRepo, cmd.IdempotencyKey, cmd, func() (*command.CreateReviewCommandResult, error) {
		// load + validate referenced aggregates; build value objects via NewX
		newReview := entities.NewReview(/* ... */)
		validated, err := entities.NewValidatedReview(newReview)
		if err != nil {
			return nil, err
		}
		if _, err := s.reviewRepository.Create(ctx, validated); err != nil {
			return nil, err
		}
		return &command.CreateReviewCommandResult{Result: mapper.NewReviewResultFromValidatedEntity(validated)}, nil
	})
}
```

The service **orchestrates, never decides** — every rule is enforced by a domain
type it invokes ([`cqrs.md`](${CLAUDE_PLUGIN_ROOT}/reference/cqrs.md)). Result mappers
produce output shapes, not entities. Write **application tests** with in-memory
fakes, including the concurrency/idempotency assertions.

### 5. Interface

- Request/response DTOs under `internal/interface/api/rest/dto/` using explicit
  primitives (`price_cents`, `currency`), plus `ToXCommand` converters. DTOs never
  expose domain types.
- `ReviewController` in `internal/interface/api/rest/` registering routes on the
  Echo instance in its constructor:
  `POST /api/v1/reviews`, `GET /api/v1/reviews`, `GET/PUT/DELETE /api/v1/reviews/:id`.
- Route write errors through the shared `writeCommandError` mapper, and add any
  new sentinel there (`ErrReviewNotFound` → 404).
- Wire it in `cmd/<service>/main.go`: construct repo → service → controller.
- If the project keeps an OpenAPI spec (`api/openapi.yaml`), add the new paths.

## Verify

1. `gofmt`/`goimports` clean; `go build ./...` and `go vet ./...` pass.
2. `make sqlc` produced the generated code the repo references (no undefined
   `db.CreateReview*` symbols).
3. Tests pass at each layer: `make test-unit` for domain+application, `make test`
   (Docker) for infrastructure. Run the idempotency/concurrency test with `-race`.
4. `make lint` clean.
5. Spot-check against **audit**'s checklist: no import-rule violation, no struct
   literal for the entity outside `entities`, `ValidatedX` on repo writes but not
   reads, every read query filters `deleted_at IS NULL`, sentinel errors wrapped
   with `%w`, `Id` (not `ID`) spelling.

State honestly which of these you actually ran versus couldn't (e.g. no Docker for
`make test`).

## Output

Produce: the new/edited files across all five layers and the migration pair, plus
a summary listing every file added/changed, the routes exposed, the invariants
encoded in `validate()`, which other aggregates it references by Id, and which
verification steps passed vs. were skipped.

## Reference

For full detail, consult the guidelines bundled with this plugin
(`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/architecture.md` — layers and the import rules the build order respects
- `reference/entities.md` — constructors, `validate()`, the `ValidatedX` pattern
- `reference/value-objects.md` — money and other values the aggregate holds
- `reference/aggregates.md` — is this really its own aggregate? boundaries, Id refs
- `reference/repositories.md` — interface-in-domain, sqlc impl, transactions, find/get
- `reference/cqrs.md` — commands, queries, services that orchestrate
- `reference/idempotency.md` — the `withIdempotency` decorator on writes
- `reference/domain-events-outbox.md` — recording + persisting events (if needed)
- `reference/testing.md` — what to test at each layer
- `reference/conventions.md` — soft deletes, defaults-in-domain, sentinels, `Id`
- `reference/tooling.md` — the sqlc/migrate workflow
