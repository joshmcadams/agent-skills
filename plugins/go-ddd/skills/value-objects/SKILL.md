---
name: value-objects
description: How to design value objects in a Go DDD service — immutable types defined only by their value, validated at construction, compared by value, with unexported fields and a single NewX constructor, plus revalidation at JSON/DB boundaries. Covers Money as integer minor units (why float64 money is a bug). Use when modeling money, an email, a percentage, a code, a quantity-with-unit, or any value that keeps getting re-validated in multiple places, or when reviewing a float64 price field.
type: knowledge
---

# Value objects, starting with Money

Design guidance for **value objects** — things defined only by their value
(`Money`, an email address), not by identity. Properties to aim for: immutable,
validated at construction, compared by value, safe to copy and pass anywhere.

> **These are opinions, not a standard**, except that money-as-integer-minor-units
> is close to non-negotiable. Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This is a **knowledge** skill — advise; don't modify files unless asked. To add a
value object to a codebase, use **add-value-object**.

## The recipe

1. **Unexported fields + one `NewX` constructor** → the constructor is the only way
   to build the type outside its package, so *every* instance is valid. For a value
   object you can make invalid states fully unrepresentable, because nothing needs
   to mutate it.
2. **Validate in the constructor**, wrapping `ErrValidation` with `%w`.
3. **Immutable** — no setters; operations return a *new* value and force you to
   decide the hard cases (what does `EUR + USD` mean? → an error; conversion is an
   explicit domain operation).
4. **Comparable by value** — keep it a plain comparable struct so `==` compares all
   components (amount *and* currency), turning a whole bug class into a type error.
5. **Revalidate at every boundary** — implement `UnmarshalJSON` to route back
   through the constructor, because JSON arrives from other services, old outbox
   rows, and hand-edited fixtures. Same for DB reconstruction (**repositories**).

```go
type Money struct { cents int64; currency Currency }

func NewMoney(cents int64, currency Currency) (Money, error) {
	if cents < 0 {
		return Money{}, fmt.Errorf("%w: amount must not be negative", ErrValidation)
	}
	if _, ok := supportedCurrencies[currency]; !ok {
		return Money{}, fmt.Errorf("%w: unsupported currency %q", ErrValidation, currency)
	}
	return Money{cents: cents, currency: currency}, nil
}
```

## Money specifically

- **Never `float64`.** Binary floating point can't represent most decimal
  fractions; errors drift under accumulation (a thousand 10-cent charges ≠ 100.00)
  and a bare number has no currency, so the type system waves 100× errors through.
- **Integer minor units (`int64` cents) + attached currency.** Exact addition and
  comparison, trivial to sort/index, ~±92 quadrillion range. (Decimal libraries are
  for FX/interest math — a different problem.)
- **Whitelist supported currencies** in the domain — adding one is a one-line
  change that forces a decision, better than accepting `"BTC"` from a client.
- At the REST edge, expose explicit primitives (`price_cents`, `currency`), never
  the domain type. The name `price_cents` removes the "19.99 or 1999?" ambiguity.
- Know the sharp edges: `String()` assuming 2 decimals breaks for JPY/KWD; division
  rounds (persist allocations, don't recompute); parse currency-aware at ingestion.

## When is something a value object?

The test: *do I keep validating this same shape in multiple places?* If yes — email
addresses, percentages, date ranges, country codes, quantities-with-units — give it
a constructor and a type and delete the scattered checks. If it has identity and a
life cycle instead, it's an entity (**entities**).

## Applying these rules

- **Designing/adding:** unexported fields, `NewX` with validation, value equality,
  `MarshalJSON`/`UnmarshalJSON` through the constructor, pure tests per rule.
- **Reviewing:** flag `float`/`float64` for money or any validated value; exported
  mutable fields on a value type; a value type that can be constructed without its
  constructor; deserialization that bypasses the constructor.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/value-objects.md` — the full treatment (float pitfalls, JSON, sharp edges)
- `reference/entities.md` — the identity-bearing counterpart
- `reference/repositories.md` — reconstruction through the constructor at the DB boundary
