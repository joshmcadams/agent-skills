---
name: add-value-object
description: Add a new value object (an immutable, value-compared type validated at construction — Money, Email, Percentage, a code, a quantity-with-unit) to a Go DDD service's domain layer, with unexported fields, a NewX constructor, value equality, and JSON (un)marshaling that routes back through the constructor. Use when the user asks to add a value object, or to replace a repeatedly-validated primitive/field (e.g. a float64 price or a bare string email) with a proper type.
type: builder
argument-hint: "<ValueObjectName>"
---

# Add a Value Object

Add one immutable value object to the domain layer so invalid states are
unrepresentable and every boundary crossing re-validates.

> **These are opinions, not a standard** (money-as-integer-minor-units aside).
> Adapted from [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This skill **modifies the codebase**. Produce the new/edited files and a summary.

## Arguments

`$ARGUMENTS` is the type name, PascalCase (`Email`, `Percentage`, `Quantity`). If
the rules that make it valid aren't stated, ask — the validation *is* the type.

## Step 0 — Orient

Read `internal/domain/entities/money.go` (or the nearest existing value object) and
mirror its idioms. Confirm this really is a **value object** (defined by value, no
identity, immutable). If it has identity and a life cycle, it's an entity — use
**add-aggregate** instead.

## Step 1 — The type, in `internal/domain/entities/<name>.go`

- **Unexported fields**, so the constructor is the only way in.
- A `NewX(...) (X, error)` constructor that validates, wrapping `ErrValidation` with
  `%w`. Add any new sentinel to `errors.go`.
- Keep it a **plain comparable struct** so `==` compares every component (don't add
  slices/maps/pointers that break comparability unless you also add an `Equal`
  method and document why).
- Exported **accessors** (`func (x X) Field() T`) for what infrastructure/DTOs need.
- Operations return **new values**, never mutate; force the hard decisions in the
  signature (e.g. mismatched-unit `Add` returns an error).

```go
type Email struct{ value string }

func NewEmail(raw string) (Email, error) {
	addr := strings.TrimSpace(strings.ToLower(raw))
	if !emailPattern.MatchString(addr) {
		return Email{}, fmt.Errorf("%w: invalid email %q", ErrValidation, raw)
	}
	return Email{value: addr}, nil
}

func (e Email) String() string { return e.value }
```

## Step 2 — Boundary (de)serialization

If the value crosses JSON or the DB, add `MarshalJSON`/`UnmarshalJSON` that route
**back through the constructor**, so a value from another service, an old outbox
row, or a hand-edited fixture can't deserialize into an invalid state. For money-like
types marshal an explicit shape (`{cents, currency}`); for simple wrappers a string.
At the REST edge, expose explicit primitives in the DTO, not the domain type.

## Step 3 — Use it

Replace the primitive it supersedes: swap `float64`/`string` fields on entities for
the value object, route construction through `NewX` in entity constructors, and
reconstruct through `NewX` in the repository's `xFromRow`. If it replaces a DB
column shape (e.g. a `price float` → `price_cents`/`currency`), that's a schema
change — use **add-migration** / **modify-schema**.

## Verify

1. `gofmt`/`goimports` clean; `go build ./...` and `go vet ./...` pass.
2. Pure domain tests (microseconds): construction rejects each invalid case
   (`assert.ErrorIs(t, err, entities.ErrValidation)`), value equality holds, and a
   JSON round-trip plus a hand-corrupted payload is rejected. See **testing**.
3. `make lint` clean.

## Output

The new value-object file, its test, any `errors.go` addition, and every call site
changed to use it — plus a summary of the invariants enforced and boundaries covered.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/value-objects.md` — the full recipe, Money, JSON boundary, sharp edges
- `reference/entities.md` — the entities that will hold this value
- `reference/repositories.md` — reconstruction through the constructor
- `reference/conventions.md` — constructors, sentinel errors
