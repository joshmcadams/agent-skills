# Tooling map

The concrete stack the reference project uses and where each concern lives. None
of it is load-bearing for the *architecture* — the point of the layering is that
these are swappable details behind interfaces the domain owns — but this is the
default set of choices that fit the patterns well.

| Concern | Tool | Where |
|---|---|---|
| HTTP | Echo v4 | `internal/interface/api/rest/` |
| DB access | pgx/v5 + [sqlc](https://sqlc.dev) | `internal/infrastructure/db/` |
| Migrations | [golang-migrate](https://github.com/golang-migrate/migrate) | `migrations/`, `migrate.go` |
| Logging | `log/slog` (structured) | throughout |
| Tests | testify + [testcontainers](https://testcontainers.com) | `*_test.go`, `internal/testhelpers/` |
| Lint | golangci-lint v2 | `.golangci.yml` |
| API contract | OpenAPI 3 | `api/openapi.yaml` |
| Ops probes | `/healthz`, `/readyz` | `internal/interface/api/rest/health_controller.go` |
| Ids | `google/uuid` (UUIDv7) | domain constructors |

## Why sqlc instead of an ORM

In a DDD codebase the row-to-aggregate mapping is a *boundary* where invariants
can leak, so you want it explicit, dumb, and reviewable. sqlc gives
compile-time-checked SQL with zero runtime reflection deciding what loads when —
no lazy loading, no cascades, no surprise queries. ORMs solve "I don't want to
write SQL"; DDD's repositories need the opposite: SQL that does exactly what the
aggregate design says and nothing more. The repository interfaces don't know sqlc
exists — swapping it out touches `internal/infrastructure/` only.

## The sqlc workflow

Schema lives in `migrations/` (golang-migrate reads the same directory), queries
in `sql/queries/*.sql` with `-- name: X :one|:many|:exec|:execrows` annotations,
and `sqlc.yaml` points at both and emits type-safe Go into
`internal/infrastructure/db/sqlc/`. Regenerate with `make sqlc` after changing
either. Key `sqlc.yaml` settings the project relies on: `sql_package: "pgx/v5"`,
`emit_interface: true` (so a `Querier` interface is generated), `emit_empty_slices:
true`, and a `uuid` → `github.com/google/uuid.UUID` type override.

## Standard Make targets

| Target | Does |
|---|---|
| `make build` | build the server binary into `./bin` |
| `make run` | run `./cmd/<service>` |
| `make test` | all tests with the race detector (needs Docker for testcontainers) |
| `make test-unit` | only tests that don't require Docker (domain + application) |
| `make lint` | golangci-lint |
| `make fmt` | gofmt + goimports |
| `make migrate-up` | apply migrations against `$DATABASE_URL` |
| `make sqlc` | regenerate sqlc code |

## Graceful lifecycle

`main.go` builds a root context cancelled on `SIGINT`/`SIGTERM`, starts the HTTP
server and the outbox relay goroutine against it, and drains in-flight requests on
shutdown with a bounded timeout. Health/readiness probes (`/healthz`, `/readyz`)
back container orchestration.
