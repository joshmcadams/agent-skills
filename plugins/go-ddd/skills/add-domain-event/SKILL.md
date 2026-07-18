---
name: add-domain-event
description: Add a domain event to an aggregate in a Go DDD service and wire it through the transactional outbox — a past-tense event struct embedding BaseEvent, recorded by the aggregate in its constructor/mutation method, and persisted in the same transaction as the aggregate write so a relay can publish it at-least-once. Use when the user asks to emit/publish a domain event (e.g. "raise a ProductPriceChanged event", "publish an OrderShipped event") so another system can react.
type: builder
argument-hint: "<AggregatePastTense>  e.g. OrderShipped"
---

# Add a Domain Event

Add one past-tense domain event, recorded by the aggregate and delivered through the
transactional outbox.

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This skill **modifies the codebase**. Produce the new/edited files and a summary.

## Before you start

Confirm the outbox actually earns its keep here: only reach for it when this state
change must reliably reach **another system**. If the reaction is in-process, just
call the function; if losing the occasional event is fine, fire-and-forget. See
**domain-events-outbox**. Also confirm the outbox table + relay exist in the repo; if
not, that infrastructure has to be scaffolded first (table migration, `insertOutbox
Events` helper, relay) — say so.

## Step 0 — Orient

Read `internal/domain/events/` (the `DomainEvent` interface, `BaseEvent`, an existing
event like `ProductCreated`) and how the repository calls `insertOutboxEvents(ctx,
qtx, agg.PullEvents())`. Mirror the idioms.

## Step 1 — The event struct

In `internal/domain/events/<aggregate>_events.go` — past-tense name, immutable, no
behavior, embedding `BaseEvent`; a `NewX` that calls `NewBaseEvent(aggregateId)`; and
`EventName()` returning a stable dotted constant:

```go
const OrderShippedEventName = "order.shipped"

type OrderShipped struct {
	BaseEvent
	TrackingCode string
}

func NewOrderShipped(orderId uuid.UUID, trackingCode string) OrderShipped {
	return OrderShipped{BaseEvent: NewBaseEvent(orderId), TrackingCode: trackingCode}
}

func (e OrderShipped) EventName() string { return OrderShippedEventName }
```

Keep the payload the minimal facts a consumer needs — it becomes the JSONB payload.

## Step 2 — Record it on the aggregate

Record the event inside the domain method where the fact becomes true (the
constructor for a `*Created`, a mutation method otherwise), via
`p.recordEvent(events.NewOrderShipped(...))`. Never build the event in the service —
the aggregate is the thing that knows its state changed. `PullEvents()` already
clears the slice so a retried save can't double-insert.

## Step 3 — Confirm the write persists it

The event only ships if the aggregate's **write path pulls and inserts events in the
same transaction**. `Create` already does this; if you added the event to an
*update* method, verify the repository's `Update` also calls `insertOutboxEvents(ctx,
qtx, agg.PullEvents())` inside its transaction — if it doesn't, add it (this is a
real, common gap). Either the row and its events commit, or neither.

## Step 4 — Consumer note

Delivery is at-least-once — document that consumers must dedupe on the event Id
(UUIDv7). The dev `Publisher` logs via slog; production swaps in the broker.

## Verify

1. `go build ./...`, `go vet ./...`, `make lint` clean.
2. Domain test: the aggregate records the event (and `PullEvents` returns it once
   then empties).
3. **Infrastructure test against real Postgres**: after the write, the outbox row
   exists; and a forced failure rolls back *both* the aggregate row and the outbox
   rows (the atomicity claim — prove it, don't assert it). See **testing**.

## Output

The event struct, the aggregate change that records it, any repository `Update`
change to persist events, and a summary: the event name/payload, where it's
recorded, and confirmation the write transaction carries it.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/domain-events-outbox.md` — events on the aggregate, the outbox, the relay, at-least-once
- `reference/repositories.md` — the transaction the outbox insert shares
- `reference/entities.md` — recording events in constructors/mutation methods
- `reference/testing.md` — proving outbox atomicity against a real DB
