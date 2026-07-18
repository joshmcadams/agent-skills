---
name: audit
description: Audit a Domain-Driven Go codebase against the go-ddd architecture and conventions (inward-pointing layer imports, entities/value objects that guard invariants, aggregates referenced by Id, sqlc repositories, CQRS, transactional outbox, idempotent commands, soft deletes) and produce a prioritized findings report. Use when the user asks to audit, review, or check a Go service for DDD/architecture compliance, or "does this follow the go-ddd patterns?".
type: knowledge
argument-hint: "[path-or-package-glob]"
---

# Audit a Go DDD Codebase

Audit the Go service in this repository against the architecture and conventions
bundled with this plugin, and produce a prioritized findings report.

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd). Some rules below
> (notably `Id` vs `ID`, sqlc) are deliberate house choices, not universal law —
> flag deviations, but weight them as *consistency* findings, not correctness bugs,
> and respect a repo that has clearly and consistently chosen otherwise. See the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This is a **knowledge** skill — audit and report; do **not** modify files unless
the user explicitly asks for fixes.

## Arguments

`$ARGUMENTS` (optional) narrows the audit to a path or package glob (e.g.
`internal/domain` or `internal/...`). If empty, audit the whole module.

## Step 1 — Orient

1. Find `go.mod` for the module path and confirm this is a Go service.
2. Locate the layers — typically `internal/domain/`, `internal/application/`,
   `internal/infrastructure/`, `internal/interface/`, with wiring in
   `cmd/*/main.go`. If the repo is **not** layered this way at all, that absence is
   itself the top finding (the architecture isn't present to violate); report what
   structure exists and stop short of nitpicking lower rules.
3. State which packages you're auditing.

Many checks below are grep-able; prefer running the greps over eyeballing, and
cite `file:line` for each finding. The example greps assume the reference layout
(`internal/domain/`, `internal/...`, `sql/queries/`, `make` targets) — adapt the
paths to the project's actual structure you found in Step 1 before running them, or
they'll silently match nothing.

## Step 2 — Check each category

Report a finding per deviation. Severity: **high** = a rule that protects
correctness (layer leak, unvalidated write, dual-write, non-atomic idempotency,
float money); **medium** = a guardrail weakened (missing soft-delete filter, DB
defaults, error strings); **low** = consistency/house-style (`ID` vs `Id`, naming).

### Layering & imports (high — the load-bearing rule)

- [ ] **Domain imports nothing outward.** In `internal/domain/`, flag any import of
  the application/infrastructure/interface packages or of a framework/driver.
  Grep: `grep -rE '"[^"]*/internal/(application|infrastructure|interface)' internal/domain/`
  and `grep -rnE '"(github.com/labstack/echo|github.com/jackc/pgx|net/http)"' internal/domain/`.
- [ ] **Application imports no infrastructure/interface/framework.**
  `grep -rE '"[^"]*/internal/(infrastructure|interface)"|labstack/echo|jackc/pgx' internal/application/`.
- [ ] **Interface imports no infrastructure.**
  `grep -rE '"[^"]*/internal/infrastructure' internal/interface/`.
- [ ] **Only `main` wires all layers.** No non-`main` package constructs concrete
  repositories *and* controllers.

### Domain modelling (high/medium)

- [ ] **Constructors, not struct literals.** Entities are built via `NewX`. Flag
  composite literals of domain entities *outside* the `entities` package and its
  tests (the repo's `xFromRow` reconstruction is the one allowed exception).
- [ ] **`validate()` exists and holds the invariants**, each wrapping `ErrValidation`
  with `%w`.
- [ ] **`ValidatedX` at write boundaries.** Repository `Create`/`Update` take
  `*entities.ValidatedX`. Grep the interfaces:
  `grep -rn 'Create\|Update' internal/domain/repositories/` — a write taking a plain
  entity is a high finding.
- [ ] **Reads return the plain entity, not `ValidatedX`.** A `FindById`/`FindAll`
  returning `*ValidatedX` is a finding — it breaks loading historical data.
- [ ] **No `float`/`float64` money.** Grep: `grep -rnE 'float(32|64)' internal/` and
  inspect anything price/amount/money-related — money must be integer minor units
  (`*_cents int64`) with a currency, ideally a `Money` value object.
- [ ] **References to other aggregates are by Id**, not embedded structs (a
  `SomethingId uuid.UUID` field, not a nested aggregate).

### Persistence & schema (high/medium)

- [ ] **Soft deletes everywhere.** Every table has `deleted_at`; every read query
  filters `deleted_at IS NULL`; `Delete` sets `deleted_at`, not `DELETE FROM`.
  Grep `sql/queries/` for `SELECT` without a `deleted_at IS NULL` clause and for
  `DELETE FROM`.
- [ ] **No DB-side defaults for identity/timestamps** (`DEFAULT gen_random_uuid()`,
  `DEFAULT now()` on `id`/`created_at`) — defaults belong in the domain
  constructor. Grep migrations for `DEFAULT`.
- [ ] **Row reconstruction routes through validating constructors** (money via
  `NewMoney`), so corrupt rows are refused.
- [ ] **Read-after-write** on `Create`/`Update`.
- [ ] **Transaction wraps aggregate write + outbox insert** together (if events
  exist).
- [ ] **Migrations are append-only pairs** (`up`/`down`), numbered, none rewritten.

### Application & CQRS (medium)

- [ ] **Writes go through `withIdempotency`** (or an equivalent atomic reservation).
  A command service method that doesn't is a finding.
- [ ] **Services orchestrate, don't decide** — no business `if` that belongs in a
  domain `validate()`.
- [ ] **Queries/results are output shapes, not entities**; commands carry primitives.

### Idempotency (high, if commands claim to be idempotent)

- [ ] **Atomic reservation** via `INSERT ... ON CONFLICT DO NOTHING` returning
  rows-affected — **not** a check-then-store (`FindByKey` then `Store`), which has
  a race. A read-before-write idempotency implementation is a high finding.
- [ ] Distinct handling for in-flight (409) vs. **key reuse with different payload**
  (422); a TTL takeover for crashed holders; cleanup on a **detached context**
  (`context.WithoutCancel`).

### Interface (medium)

- [ ] **Sentinel errors mapped to status codes** via `errors.Is` — not string
  matching on error messages. Grep for `err.Error() ==` / `strings.Contains(err`.
- [ ] **Infrastructure errors don't leak** (no pgx types above infrastructure).
- [ ] **DTOs use explicit primitives** (`price_cents`) and don't expose domain types.

### Testing (medium/low)

- [ ] **Domain tests exist and are pure** (no DB/framework); assert sentinels
  (`assert.ErrorIs`), not message strings.
- [ ] **Repository tests hit a real database** (testcontainers), not a mocked DB.
- [ ] **Concurrency test for idempotency** run with `-race`.

### Conventions (low — consistency)

- [ ] **`Id`, not `ID`**, applied consistently. Grep: `grep -rnE '\bID\b' internal/`
  — but report only if the codebase is otherwise `Id` (respect a consistent `ID`
  codebase).
- [ ] **Ubiquitous language** — one term per concept across struct/table/route.

## Step 3 — Report

Output a Markdown report:

1. **Summary** — packages audited, counts by severity, and a one-line verdict.
2. **Findings** grouped by category, most severe first. Each finding: the rule,
   severity, `file:line`, what's wrong, and the concrete fix.
3. **Quick wins** vs **larger refactors** called out separately.

Do not modify files during an audit. To apply fixes, walk the relevant builder
(**add-aggregate** for missing end-to-end wiring) or hand-edit per the finding,
and re-run this audit.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/architecture.md` — the import rules (layering findings)
- `reference/entities.md`, `reference/value-objects.md`, `reference/aggregates.md` — domain findings
- `reference/repositories.md` — persistence, transactions, find/get, sentinels
- `reference/cqrs.md`, `reference/idempotency.md` — application findings
- `reference/domain-events-outbox.md` — dual-write / outbox findings
- `reference/testing.md` — per-layer test findings
- `reference/conventions.md` — soft deletes, defaults, `Id`, migrations
- `reference/index.md` — chapter map
