# Go DDD — Reference

Distilled, rule-style reference for building Domain-Driven Go backend services,
adapted from [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd). The
skills in this plugin cite these chapters; each is self-contained enough to act
on without the others.

These are **opinions, not a standard** — see [`NOTICE`](../NOTICE) for what that
means and for attribution.

## Chapter map

| Chapter | Covers |
|---|---|
| [`architecture.md`](architecture.md) | The onion, the four layers, the import rules that keep dependencies pointing inward, and the request flows. |
| [`entities.md`](entities.md) | Constructors over struct literals, invariants in `validate()`, the `ValidatedX` pattern, mutation through methods, sentinel errors. |
| [`value-objects.md`](value-objects.md) | Immutable values validated at construction; `Money` as integer minor units; revalidating at JSON/DB boundaries. |
| [`aggregates.md`](aggregates.md) | Consistency boundaries drawn from invariants, referencing other aggregates by Id, keeping aggregates small. |
| [`repositories.md`](repositories.md) | Interface in the domain, sqlc implementation in infrastructure; one per aggregate root; `find` vs `get`; read-after-write; transactions. |
| [`cqrs.md`](cqrs.md) | Commands and queries as explicit types; services orchestrate but don't decide; results are outputs, not entities. |
| [`domain-events-outbox.md`](domain-events-outbox.md) | Events recorded by the aggregate; the transactional outbox; the relay; at-least-once delivery. |
| [`idempotency.md`](idempotency.md) | Idempotency as a write-side claim: atomic reservation, TTL takeover, detached-context cleanup. |
| [`testing.md`](testing.md) | The per-layer testing pyramid: pure domain tests, fake-backed application tests, testcontainers for infrastructure. |
| [`conventions.md`](conventions.md) | The cross-cutting house rules: `Id` not `ID`, soft deletes, sentinel errors, append-only migrations, defaults in the domain, ubiquitous language. |
| [`tooling.md`](tooling.md) | The concrete stack (Echo, pgx/sqlc, golang-migrate, slog, testify/testcontainers, golangci-lint) and where each concern lives. |

## The one-paragraph version

Put every business rule in the domain layer, make it impossible to reach
persistence without passing validation (the `ValidatedX` types enforce this at
compile time), and keep the domain free of any framework or driver so the rules
survive every infrastructure change. Writes flow through the full model
(command → service → domain → repository) and are idempotent; reads have their
own thin path with no business rules; state changes announce themselves through
domain events delivered via a transactional outbox. The database is a detail
behind interfaces the domain owns.
