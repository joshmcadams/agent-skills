---
name: idempotency
description: How to make write commands idempotent in a Go DDD service so client retries don't create duplicates — idempotency as a write-side claim (atomic INSERT ... ON CONFLICT DO NOTHING branching on rows-affected), not a read-then-write check that races; distinguishing in-flight (409) from key-reuse-with-different-payload (422); a TTL takeover for crashed holders; and detached-context (context.WithoutCancel) cleanup. Use when adding idempotency keys, reviewing an idempotency implementation for races, or handling duplicate POSTs from retries.
type: knowledge
---

# Idempotent commands

Guidance for surviving client retries without duplicate writes. Retries are not
exotic — flaky networks, rolling deploys, and proxy timeouts guarantee that
"the server did the work" and "the client learned about it" eventually diverge.

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This is a **knowledge** skill — advise; don't modify files unless asked. The
`withIdempotency` decorator is applied by the write use cases (**cqrs**,
**add-command**).

## The one sentence that matters

**Idempotency is a write-side claim, not a read-side check.** If the implementation
reads before it writes, it has a race, full stop. The naive check-execute-store
passes every *sequential* test and fails exactly when it matters: two requests with
the same key arrive together, both `FindByKey` → `nil`, both execute. And
concurrent duplicates are the *main* case — retries fire because something is slow,
and when something is slow the original is usually still running.

## The correct implementation

1. **Reserve the key atomically**, inserting a row *before* execution as a
   reservation (not after, as a cache entry):

   ```sql
   -- name: ReserveIdempotencyKey :execrows
   INSERT INTO idempotency_records (id, key, request, response, status_code, created_at)
   VALUES ($1, $2, $3, '', 0, $4)
   ON CONFLICT (key) DO NOTHING;
   ```

   `:execrows` returns the row count: **1 = you won, may execute; 0 = someone holds
   it, must not.** Postgres's unique index is the arbiter — no Redis, no lock library.

2. **Answer the losers by the winner's state:** winner finished → serve the stored
   response (byte-for-byte); still running → `ErrRequestInFlight` → **409**; same key
   *different payload* → `ErrIdempotencyKeyReuse` → **422** (never silently return
   A's response to B); reservation past TTL with no response → the holder crashed,
   take it over.

3. **TTL takeover** so a crash between reserve and store doesn't brick the key
   forever. Size `reservationTTL` comfortably above your slowest *legitimate*
   execution (a minute for CRUD; batch jobs need a different mechanism).

4. **Release/store on a detached context.** Execution often fails *because* the
   client disconnected and cancelled the request context — so cleanup uses
   `context.WithoutCancel(ctx)` (keeps trace Ids/loggers, drops cancellation), or
   the release DELETE never reaches Postgres and the retrying client is locked out
   for a full TTL. Storing the response is best-effort (log on failure; the business
   op already committed).

One generic decorator wraps every command: `withIdempotency[T any](ctx, repo, key,
cmd, execute)`.

## Prove it under concurrency

Sequential tests can't catch the race. Fire concurrent goroutines at one key with
`-race` and assert `executions == 1` and `callers == successes + inFlight`. These
tests belong in the application layer against fast fakes (**testing**).

## Applying these rules

- **Designing:** atomic reserve branching on rows-affected; the four loser
  outcomes; TTL takeover; detached-context cleanup; a concurrency test.
- **Reviewing:** flag any `FindByKey`-then-`Store` (read-before-write race); missing
  422 for payload mismatch; no TTL takeover; cleanup on the cancellable request
  context; missing `-race` concurrency test.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/idempotency.md` — the full treatment with the decorator code
- `reference/cqrs.md` — where the decorator wraps the command
- `reference/domain-events-outbox.md` — the outbound reliability counterpart
- `reference/testing.md` — the concurrency test that proves it
