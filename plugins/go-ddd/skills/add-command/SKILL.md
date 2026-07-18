---
name: add-command
description: Add a new write use case (command) to an existing aggregate in a Go DDD service — a command struct carrying primitives plus an idempotency key, a result output type, and a service method wrapped in withIdempotency that loads/validates aggregates, invokes domain behavior, persists, and maps to a result. Use when the user asks to add a write/mutating operation to an existing aggregate (e.g. "add a CancelOrder command", "add an operation to archive a product") without adding a whole new aggregate.
type: builder
argument-hint: "<Verb><Aggregate>  e.g. CancelOrder"
---

# Add a Command (write use case)

Add one write use case to an aggregate that already exists: the command type, its
result, and the service method — orchestrating domain behavior, never deciding it.

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This skill **modifies the codebase**. Produce the new/edited files and a summary.

## Arguments

`$ARGUMENTS` names the intent in the ubiquitous language, PascalCase
(`CancelOrder`, `ArchiveProduct`). If the aggregate doesn't exist yet, use
**add-aggregate**. If the behavior is a *new business rule*, it belongs on the
entity — add the domain method first (**entities**), then this command to invoke it.

## Step 0 — Orient

Read an existing command end-to-end: the struct in `internal/application/command/`,
the service method in `internal/application/services/`, and how the controller calls
it. Mirror the idioms.

## Step 1 — Domain behavior first

If the command does more than plain field updates, add/confirm the behavior on the
entity as a method that ends in `return e.validate()` (e.g. `Cancel()`, `Archive()`)
and records a domain event if other systems care (**domain-events-outbox**). The
service must not encode the rule itself.

## Step 2 — Command + result

In `internal/application/command/`:

```go
type CancelOrderCommand struct {
	IdempotencyKey string
	Id             uuid.UUID
	// raw primitives only — no domain value objects
}

type CancelOrderCommandResult struct { Result *common.OrderResult } // or Success bool
```

## Step 3 — Service method, wrapped in idempotency

Add the method to the service and its interface in
`internal/application/interfaces/`. Wrap the body in `withIdempotency`; inside:
load the aggregate (`FindById`; nil → the not-found sentinel), load+validate any
referenced aggregates, invoke the domain method, re-wrap as `ValidatedX`, persist
via the repository, and map to a result.

```go
func (s *OrderService) CancelOrder(ctx context.Context, cmd *command.CancelOrderCommand) (*command.CancelOrderCommandResult, error) {
	return withIdempotency(ctx, s.idempotencyRepo, cmd.IdempotencyKey, cmd, func() (*command.CancelOrderCommandResult, error) {
		order, err := s.orderRepository.FindById(ctx, cmd.Id)
		if err != nil { return nil, err }
		if order == nil { return nil, entities.ErrOrderNotFound }
		if err := order.Cancel(); err != nil { return nil, err }
		validated, err := entities.NewValidatedOrder(order)
		if err != nil { return nil, err }
		if _, err := s.orderRepository.Update(ctx, validated); err != nil { return nil, err }
		return &command.CancelOrderCommandResult{Result: mapper.NewOrderResultFromValidatedEntity(validated)}, nil
	})
}
```

If the command needs a repository access pattern that doesn't exist yet (e.g.
`FindByStatus`), add it with **add-repository-method** first.

## Step 4 — Expose it (if it has an endpoint)

Add the request DTO + `ToXCommand` converter, a controller handler and route, and a
`writeCommandError` mapping for any new sentinel — or, if it's internal-only, wire
it wherever it's invoked. DTOs use explicit primitives and never expose domain types.

## Verify

1. `go build ./...`, `go vet ./...`, `make lint` clean.
2. Application test with in-memory fakes: happy path, not-found, validation failure,
   and the idempotency/concurrency behavior (`-race`) — see **testing** and
   **idempotency**.

## Output

The new command/result, the service method (+ interface entry), any domain method
and endpoint wiring, and a summary: the intent, the invariants it relies on, which
aggregates it touches, and which verification steps passed.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/cqrs.md` — commands, services that orchestrate, results
- `reference/idempotency.md` — the `withIdempotency` wrapper
- `reference/entities.md` — the domain method the command invokes
- `reference/repositories.md` — loading and persisting the aggregate
