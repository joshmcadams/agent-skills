# Idempotent commands

A client sends `POST /products`. The server creates it, then the load balancer
kills the connection before the response escapes. The client sees a timeout and
retries. Now you have two products. This is not exotic — it's Tuesday: flaky
mobile networks, rolling deploys, proxy timeouts. The fix is an **idempotency
key**: the client sends a unique key per logical operation, and the server
guarantees the same key never executes the operation twice.

## The one-sentence version

**Idempotency is a write-side claim, not a read-side check.** If your
implementation reads before it writes, it has a race, full stop. Make the
database's unique constraint do the deciding and branch on rows-affected.
Everything below is a consequence of taking that sentence seriously.

## The naive version has a race

```go
existing, _ := repo.FindByKey(ctx, key)   // both concurrent requests get nil
if existing != nil { return cached(existing) }
result, _ := doWork(ctx)                    // both execute
repo.Store(ctx, key, result)                // two products, one key
```

Check-execute-store reads correctly and passes every *sequential* test — which is
exactly why the bug survives. Two requests with the same key arriving at the same
time both see `nil` and both execute. And concurrent duplicates are the **main**
case: retries fire because something is slow, and when something is slow the
original request is usually still running.

## Fix 1 — reserve the key atomically

The check and the claim must be one atomic operation: a unique constraint plus
`ON CONFLICT DO NOTHING`, inserting the row *before* execution as a reservation
(not after, as a cache entry):

```sql
-- name: ReserveIdempotencyKey :execrows
-- Atomically claims the key. Zero rows means another request already holds it.
INSERT INTO idempotency_records (id, key, request, response, status_code, created_at)
VALUES ($1, $2, $3, '', 0, $4)
ON CONFLICT (key) DO NOTHING;
```

`:execrows` returns the affected-row count, and that count is the whole trick:
**1 = you won the race and may execute; 0 = someone else holds the key and you
must not.** Postgres's unique index is the arbiter — no advisory locks, no Redis,
no distributed-lock library.

## The losers need answers — four outcomes

If you lost the race, what you tell the client depends on the winner's state:

- **Winner finished** → serve its stored response (the same `201` body, byte for
  byte).
- **Winner still running** → `ErrRequestInFlight`, mapped to **409 Conflict**.
  Back off, retry, hit the cached response.
- **Same key, different payload** → `ErrIdempotencyKeyReuse`, mapped to **422**.
  Dangerous to skip: silently returning request A's cached response to request B
  means the caller thinks it created B while holding A. Fail loudly.
- **Reservation older than the TTL with no response** → the holder is dead; take
  it over (Fix 2).

One generic decorator wraps every command identically:

```go
func withIdempotency[T any](ctx context.Context, repo repositories.IdempotencyRepository,
	key string, cmd any, execute func() (*T, error)) (*T, error) {
	if key == "" {
		return execute()
	}
	// marshal cmd → requestJSON; build record
	reserved := false
	for attempt := 0; attempt < 3 && !reserved; attempt++ {
		reserved, err = repo.Reserve(ctx, record)
		if err != nil { return nil, err }
		if reserved { break }

		existing, err := repo.FindByKey(ctx, key)
		if err != nil { return nil, err }
		if existing == nil { continue } // released between Reserve and FindByKey

		if existing.Request != string(requestJSON) { return nil, ErrIdempotencyKeyReuse }
		if existing.IsCompleted() { /* unmarshal + return cached */ }
		if time.Since(existing.CreatedAt) < reservationTTL { return nil, ErrRequestInFlight }

		// Stale reservation: previous holder crashed. Release and retry.
		if err := repo.Delete(ctx, key); err != nil { return nil, err }
	}
	// ... execute + release-on-failure + store response (below)
}
```

## Fix 2 — crashes must not brick a key

Reserving before executing creates a new failure mode: reserve, then die (OOM,
deploy) before storing a response. The row says "in flight" forever and every
retry gets 409. So a reservation past `reservationTTL` with no response means the
holder is dead; the next retry deletes the stale row and re-reserves. Because
re-reserving is the same atomic `INSERT ... ON CONFLICT`, two retries racing to
take over still resolve to one winner.

Size the TTL comfortably longer than your slowest *legitimate* execution, or
you'll take over merely-slow requests and be back to duplicates. A minute is
generous for CRUD; a batch job needs a different mechanism.

## Fix 3 — release on failure with a detached context

On failure, release the reservation so the retry can run. But execution often
fails *because* the client disconnected and the request context was cancelled —
call `repo.Delete(ctx, key)` with that cancelled context and the DELETE never
reaches Postgres. The reservation leaks and the disconnecting client (the one most
likely to retry) is locked out for a full TTL. `context.WithoutCancel` (Go 1.21+)
keeps values (trace Ids, loggers) but detaches cancellation:

```go
result, err := execute()
if err != nil {
	if deleteErr := repo.Delete(context.WithoutCancel(ctx), key); deleteErr != nil {
		slog.WarnContext(ctx, "failed to release idempotency key",
			slog.String("idempotency_key", key), slog.Any("error", deleteErr))
	}
	return nil, err
}
storeResponse(context.WithoutCancel(ctx), repo, key, result)
```

Storing the response is **best-effort** for the same reason: the business
operation already committed, so persisting the cache entry must not be at the
mercy of the request context, and a failed cache write must not fail an
already-successful operation. Log it; the TTL takeover covers the wedged row.

## Prove it under concurrency

Sequential tests can't catch the original bug. Fire concurrent goroutines at one
key and assert the only thing that matters — run it with `-race`:

```go
assert.Equal(t, 1, executions, "exactly one caller may execute")
assert.Equal(t, callers, successes+inFlight)
```

Successes can exceed one (a loser arriving after the winner finished legitimately
gets the cached response), but the business logic runs exactly once, always. These
tests belong in the application layer against fast fakes ([`testing.md`](testing.md)),
where you can afford thousands of runs.
