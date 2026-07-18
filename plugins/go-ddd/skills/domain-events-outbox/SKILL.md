---
name: domain-events-outbox
description: How to publish domain events reliably from a Go DDD service using the transactional outbox pattern — the aggregate records past-tense events, the aggregate write and the outbox insert share one transaction (solving the dual-write problem), and a relay polls unpublished rows and publishes them at-least-once. Use when a state change must reliably reach another system/broker, when choosing between publishing directly vs an outbox, or when designing domain events, the outbox table, or the relay. Also covers when to skip the outbox.
type: knowledge
---

# Domain events and the transactional outbox

Guidance for announcing state changes reliably. The problem it solves is the
**dual-write problem**: the database and the broker are two systems and no
transaction spans both, so "save then publish" silently drifts (crash after commit
→ lost event) or produces ghost events (publish then failed insert).

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This is a **knowledge** skill — advise; don't modify files unless asked. To add an
event to an aggregate and wire it through the outbox, use **add-domain-event**.

## The pattern

1. **The aggregate records events**, not the service. "A product was created" is a
   domain fact the aggregate knows. The aggregate appends events in its constructor
   / mutation methods and exposes them via `PullEvents()`, which *clears* the slice
   so a retried save can't double-insert.
2. **Events are dumb structs** — past-tense names, immutable, no behavior — behind a
   `DomainEvent` interface (`EventId`, `EventName`, `OccurredAt`, `AggregateId`). A
   shared `BaseEvent` carries the common fields; the event Id is a **UUIDv7**,
   time-ordered and usable as a consumer dedup key.
3. **One transaction or it didn't happen.** In the repository, the aggregate insert
   and the outbox insert share one pgx transaction (`insertOutboxEvents(ctx, qtx,
   agg.PullEvents())`). Either both commit or neither does. The service knows
   nothing about it — **you can't forget to publish, because there's no publish step
   to forget.**
4. **A partial index keeps the relay query cheap forever:** `CREATE INDEX ... ON
   outbox_events(occurred_at) WHERE published_at IS NULL`. The table grows forever;
   the index stays tiny.
5. **A dumb relay** polls unpublished rows, hands each to a `Publisher` interface
   (log in dev; Kafka/NATS/SQS in prod), and marks it published. Delivery is
   **at-least-once** (mark-published can fail after a successful publish → redeliver
   next tick), so **every consumer must be idempotent**, deduping on the event Id.

## Caveats to state when advising

- **Ordering** is per-relay by `occurred_at` (one aggregate's events in order);
  partition a topic by `aggregate_id`; never promise cross-aggregate ordering.
- **Scaling the relay** — two naive instances double-publish; add `FOR UPDATE SKIP
  LOCKED` to the poll query when you measure the need.
- **Polling vs CDC** — start with polling (zero extra infra); reach for Debezium/WAL
  only for low-latency needs.
- **Cleanup** — a nightly `DELETE ... WHERE published_at < now() - interval '30
  days'` keeps the table sane.

## When to skip the outbox

Often. If the handler lives in the same process (creating a product warms a cache),
just call the function — in-process, in-transaction, done. If losing the occasional
event is tolerable (analytics pings), fire-and-forget is fine. Reach for the outbox
exactly when a state change in *your* database must reliably reach *another* system:
"when X happens here, Y must happen over there."

## Applying these rules

- **Designing:** record events on the aggregate; persist them in the write
  transaction; keep the relay and publisher dumb; make consumers idempotent.
- **Reviewing:** flag a service that publishes directly to a broker after a separate
  DB commit (dual-write); events built in the service instead of the aggregate;
  outbox inserts not in the aggregate's write transaction; consumers assuming
  exactly-once.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/domain-events-outbox.md` — the full treatment with table, relay, caveats
- `reference/repositories.md` — the transaction the outbox insert rides in
- `reference/idempotency.md` — the inbound counterpart (idempotent commands)
- `reference/aggregates.md` — events as the integration mechanism between contexts
