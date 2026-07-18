# Testing a DDD codebase

The layering decides the testing strategy for you. Each layer's tests answer a
different question, and none re-answers a lower layer's question.

| Layer | What's tested | Against | Speed |
|---|---|---|---|
| Domain | Business rules, invariants, events | Nothing — pure Go | microseconds |
| Application | Orchestration, idempotency, error flows | In-memory fakes | milliseconds |
| Infrastructure | SQL, mapping, transactions | **Real Postgres** (testcontainers) | seconds |

## Domain tests — pure functions in disguise

The domain has no dependencies — no database, no framework, no clock injection.
Its tests are plain table-stakes Go that read like the rules they verify:

```go
func TestNewMoney_RejectsNegativeAmount(t *testing.T) {
	_, err := entities.NewMoney(-1, entities.EUR)
	assert.ErrorIs(t, err, entities.ErrValidation)
}

func TestProduct_PullEventsClearsEvents(t *testing.T) {
	product := entities.NewProduct("Widget", price, validatedSeller)
	first := product.PullEvents()
	second := product.PullEvents()
	assert.Len(t, first, 1)
	assert.Empty(t, second)
}
```

Two rules: assert against **sentinel errors** (`assert.ErrorIs(t, err,
ErrValidation)`), never message strings — messages are documentation, sentinels
are contract. And the *whole invariant surface* of an entity is testable without
starting anything — the payoff of keeping rules in the entity. These run in
microseconds, so they run constantly; there's no excuse for an untested business
rule when the test costs this little.

## Application tests — fakes, not mock frameworks

Because repositories are interfaces owned by the domain
([`repositories.md`](repositories.md)), services take simple in-memory fakes:

```go
type MockProductRepository struct {
	products []*entities.ValidatedProduct
}

func (m *MockProductRepository) Create(ctx context.Context, product *entities.ValidatedProduct) (*entities.Product, error) {
	m.products = append(m.products, product)
	return &product.Product, nil
}
```

A slice and some loops — no `EXPECT().Times(1)` choreography. Prefer hand-written
fakes here for a concrete reason: expectation-style mocks test *how* the service
talks to its dependencies, welding tests to the implementation; fakes test *what
ends up true*, which survives refactors (the GORM → sqlc migration didn't change
these tests). Concurrency tests — the idempotency suite firing goroutines at one
key and asserting `executions == 1` — belong here, against fast fakes with
`-race`, not against a container where each attempt costs seconds.

## Infrastructure tests — a real database or it doesn't count

The strong opinion: **mocking the database in repository tests is worthless.** The
repository's entire job is SQL, transactions, and row mapping; a test that mocks
the SQL away only verifies the code calls the mock. Every real repository bug
lives in the parts a mock skips — a query that ignores `deleted_at`, a mapping
that flips two columns, a transaction that doesn't actually cover the outbox
insert. So repository tests run against real Postgres via
[testcontainers](https://testcontainers.com):

```go
func SetupTestDB(t *testing.T) *PostgresTestContainer {
	postgresContainer, err := postgres.Run(ctx, "postgres:17-alpine",
		postgres.WithDatabase(dbName), postgres.WithUsername(dbUser), postgres.WithPassword(dbPassword),
		testcontainers.WithWaitStrategy(
			wait.ForLog("database system is ready to accept connections").WithOccurrence(2)),
	)
	require.NoError(t, err, "Failed to start postgres container")
	// connect, apply migrations, return pool + queries
}
```

Each test package gets a disposable Postgres in Docker: real unique constraints
(the idempotency reservation *depends* on one), real transaction semantics (the
outbox atomicity claim is *tested*, not asserted), real `ON CONFLICT` behavior.
The cost is seconds of startup per package, paid honestly in CI on every push and
locally via `make test`. What you get is the class of confidence mocks can't sell:
proving that a failed aggregate insert rolls back the outbox rows too.

## What deliberately isn't tested

- **No tests that a fake returns what it was told to return.** If a test can only
  fail when its own setup changes, delete it.
- **No E2E test for every feature.** One smoke path (bring the stack up, create a
  seller and product, watch the relay fire) covers the wiring; feature behavior is
  already covered below, faster.
- **No coverage worship.** Generated code (sqlc output) is excluded from the
  metric. Chasing a number through generated files buys dashboards, not tested
  invariants.

## The feedback loop is the point

Rule change → domain test, microseconds. New orchestration → fake-backed test,
milliseconds. New query → one container-backed test, seconds. Each change lands in
the cheapest layer that can catch its bugs — and the architecture is what made
those layers separable. A codebase where business rules are testable in
microseconds *without Docker* is a codebase where the rules actually get tested.
