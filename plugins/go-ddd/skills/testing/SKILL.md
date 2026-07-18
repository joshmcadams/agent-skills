---
name: testing
description: How to test a Go DDD codebase so tests are fast where they can be and honest where they must be — pure microsecond domain tests (asserting sentinel errors, not strings), millisecond application tests against hand-written in-memory fakes (not mock frameworks), and infrastructure/repository tests against a real Postgres via testcontainers (never a mocked DB). Use when deciding what to test at which layer, choosing fakes vs mocks, whether a repository test needs a real database, or where a concurrency test belongs.
type: knowledge
---

# Testing a DDD codebase

The layering decides the testing strategy: each layer's tests answer a different
question against a different backing, and none re-answers a lower layer's question.

| Layer | Tests | Against | Speed |
|---|---|---|---|
| Domain | rules, invariants, events | nothing — pure Go | microseconds |
| Application | orchestration, idempotency, error flows | in-memory fakes | milliseconds |
| Infrastructure | SQL, mapping, transactions | **real Postgres** (testcontainers) | seconds |

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This is a **knowledge** skill — advise; don't modify files unless asked.

## Domain tests — pure functions

No database, no framework, no clock injection. Plain table-stakes Go that reads like
the rules. Two disciplines: assert against **sentinel errors**
(`assert.ErrorIs(t, err, entities.ErrValidation)`), never message strings (messages
are documentation, sentinels are contract); and cover the *whole invariant surface*
of each entity — it's testable without starting anything, so there's no excuse for
an untested rule. These run in microseconds and run constantly.

## Application tests — fakes, not mock frameworks

Because repositories are domain-owned interfaces, hand services simple in-memory
fakes (a slice + a mutex), not `EXPECT().Times(1)` choreography. Reason:
expectation-mocks test *how* the service talks to its dependencies, welding tests to
the implementation; fakes test *what ends up true*, which survives refactors (a
GORM→sqlc migration leaves them unchanged). **Concurrency tests belong here** — the
idempotency suite firing goroutines at one key, asserting `executions == 1`, run
with `-race` — against fast fakes, not a container costing seconds per attempt.

## Infrastructure tests — a real database or it doesn't count

**Mocking the database in repository tests is worthless** — the repository's whole
job is SQL, transactions, and row mapping, and a mock only proves the code calls the
mock. Every real repository bug lives where a mock skips: a query ignoring
`deleted_at`, a mapping flipping two columns, a transaction not covering the outbox
insert. Run repository tests against a disposable Postgres via testcontainers, so
unique constraints (idempotency depends on one), transaction semantics (outbox
atomicity), and `ON CONFLICT` behavior are *real*. Costs seconds per package, paid
honestly in CI (`make test`) — `make test-unit` skips the Docker-backed ones.

## What not to test

- No test that a fake returns what it was told to return (fails only when its own
  setup changes → delete it).
- No E2E test per feature — one smoke path covers the wiring; behavior is covered
  faster below.
- No coverage worship — exclude generated code (sqlc output) from the metric.

## Applying these rules

- **Designing:** push each test to the cheapest layer that can catch its bug — rule
  → domain (µs), orchestration → fake-backed (ms), query/mapping/transaction →
  container (s).
- **Reviewing:** flag mocked databases in repository tests; expectation-style mocks
  where a fake would do; assertions on error strings; concurrency tests against
  containers instead of fakes; invariants with no domain test.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/testing.md` — the full treatment with per-layer examples
- `reference/repositories.md` — the interfaces that make fakes trivial
- `reference/idempotency.md` — the concurrency suite
- `reference/conventions.md` — sentinel errors that tests assert against
