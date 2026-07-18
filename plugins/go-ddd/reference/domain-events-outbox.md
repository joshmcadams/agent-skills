# Domain events and the transactional outbox

The problem: save to Postgres, then publish to a broker. It works in every demo
and ~99.9% of the time in production. The 0.1% is where the fun is — the database
and the broker are two systems and **no transaction spans both**:

- Crash after the DB commit, before the publish → the row exists, but no consumer
  ever hears about it. Silent data drift.
- Publish first, then the insert fails → a *ghost event*: consumers react to
  something that was never created.

You can't "be careful" your way out; a retry loop shrinks the window, never closes
it. This is the **dual-write problem**, and the fix is the **transactional
outbox**: don't publish the event — *store* it, in the same database, in the same
transaction as the state change. A separate process publishes it.

## Decision 1 — events belong to the aggregate

Who creates the event? Not the service. "A product was created" is a domain fact,
and the aggregate is the thing that knows its own state changed. So the aggregate
records events as part of the change, and exposes them via a pull-once method:

```go
func NewProduct(name string, price Money, seller ValidatedSeller) *Product {
	product := &Product{ /* ... */ }
	product.recordEvent(events.NewProductCreated(
		product.Id, name, price.Cents(), string(price.Currency()), seller.Id))
	return product
}

// PullEvents returns the recorded events and clears them, so a retried save
// can't double-insert the same events.
func (p *Product) PullEvents() []events.DomainEvent {
	pulled := p.domainEvents
	p.domainEvents = nil
	return pulled
}
```

Events are dumb structs — past-tense names, immutable, no behavior. A shared
`BaseEvent` carries the common fields; the concrete event adds its payload and
implements `EventName`:

```go
type DomainEvent interface {
	EventId() uuid.UUID
	EventName() string
	OccurredAt() time.Time
	AggregateId() uuid.UUID
}

const ProductCreatedEventName = "product.created"

func (e ProductCreated) EventName() string { return ProductCreatedEventName }
```

The event Id is a UUIDv7 — time-ordered, so it sorts nicely and doubles as a
**deduplication key** for consumers.

## Decision 2 — one transaction or it didn't happen

The outbox table is deliberately simple, with a **partial index** on unpublished
rows:

```sql
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    event_name TEXT NOT NULL,
    payload JSONB NOT NULL,
    occurred_at TIMESTAMP WITH TIME ZONE NOT NULL,
    published_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_outbox_events_unpublished
    ON outbox_events(occurred_at) WHERE published_at IS NULL;
```

The partial index matters: the relay only ever asks "unpublished events, oldest
first," while the table grows forever. An index `WHERE published_at IS NULL` stays
tiny no matter how many millions of published rows pile up.

The aggregate insert and the outbox insert share one transaction in the repository
([`repositories.md`](repositories.md)):

```go
qtx := repo.queries.WithTx(tx)
if _, err := qtx.CreateProduct(ctx, /* ... */); err != nil {
	return nil, err
}
if err := insertOutboxEvents(ctx, qtx, product.PullEvents()); err != nil {
	return nil, err
}
// ... read-back, then tx.Commit(ctx)
```

Either the row *and* its events commit, or neither does. The service knows nothing
about this — **you can't forget to publish, because there is no publish step to
forget.**

## The relay — dumb on purpose

A loop polls unpublished rows and hands each to a `Publisher`, then marks it
published:

```go
type Publisher interface {
	Publish(ctx context.Context, eventName string, payload []byte) error
}
```

In the reference project the publisher just logs via `slog`; in production you
swap in Kafka, NATS, SQS. The failure mode is the contract: publish succeeds, then
the mark-published write fails → next tick, the row is still unpublished, so it
publishes **again**. The outbox gives **at-least-once** delivery, never
exactly-once. **Every consumer must be idempotent** — the event Id is the dedup
key. (You needed idempotent consumers anyway; brokers redeliver on their own.)

## Caveats the diagram omits

- **Ordering** is per-relay, by `occurred_at` — one aggregate's events come out in
  recording order. For a partitioned topic, partition by `aggregate_id`. Never
  promise cross-aggregate ordering.
- **Scaling the relay** — two naive instances double-publish. The fix is
  `FOR UPDATE SKIP LOCKED` in the poll query; add it when you measure the need.
- **Polling vs CDC** — polling every few seconds needs zero extra infrastructure
  and covers a huge range of workloads. Start there; reach for Debezium/WAL
  tailing only for low-latency needs.
- **Cleanup** — published rows pile up; a nightly `DELETE ... WHERE published_at <
  now() - interval '30 days'` keeps the table sane.

## When to skip the outbox

Often. If the "event handler" lives in the same process (creating a product warms
a cache), just call the function — in-process, in-transaction, done. If losing the
occasional event is tolerable (analytics pings), fire-and-forget is fine. The
outbox earns its keep exactly when a state change in *your* database must reliably
reach *another* system. The moment someone says "when X happens here, Y must
happen over there" — reach for it.
