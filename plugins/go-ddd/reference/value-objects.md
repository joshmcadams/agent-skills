# Value objects, starting with Money

Entities have identity; **value objects** have only value. Two `Money` values of
19.99 EUR are the same value, the way two 7s are the same number. That definition
buys concrete properties: value objects are **immutable, validated at
construction, compared by value, and safe to copy and pass anywhere**.

`Money` is the canonical example because getting it wrong is expensive in the
literal sense, and it's the most common value-object bug in the wild.

## Why `float64` money is broken, not just risky

Binary floating point can't represent most decimal fractions exactly. The version
that hurts is drift under accumulation — a thousand 10-cent charges and
`total == 100` is already `false`. Feed that into a threshold check or a
reconciliation diff and you're in epsilon-comparison whack-a-mole. And a bare
`float64` has a second problem: `19.99` *of what*? EUR? USD? cents already? The
type system waves a 100× pricing error straight through.

## The Money value object

Integer minor units, currency attached, **both fields unexported**:

```go
type Currency string

const (
	EUR Currency = "EUR"
	USD Currency = "USD"
)

var supportedCurrencies = map[Currency]struct{}{EUR: {}, USD: {}}

type Money struct {
	cents    int64
	currency Currency
}

func NewMoney(cents int64, currency Currency) (Money, error) {
	if cents < 0 {
		return Money{}, fmt.Errorf("%w: amount must not be negative", ErrValidation)
	}
	if _, ok := supportedCurrencies[currency]; !ok {
		return Money{}, fmt.Errorf("%w: unsupported currency %q", ErrValidation, currency)
	}
	return Money{cents: cents, currency: currency}, nil
}

func (m Money) Cents() int64       { return m.cents }
func (m Money) Currency() Currency { return m.currency }
```

The design decisions:

- **Unexported fields, one constructor.** Because `cents` and `currency` are
  unexported, `NewMoney` is the only way to build a `Money` outside the package.
  Every `Money` in the system has passed the checks — there is no "construct raw,
  validate later" path to forget. For a value object you *can* make invalid states
  fully unrepresentable, because nothing needs to mutate it.
- **Integer minor units, not `float64` or a decimal library.** `int64` cents give
  exact addition and comparison for free, sort and index trivially, and cover
  ~±92 quadrillion. (For FX rates or interest math, reach for a decimal type —
  a different problem.)
- **Currency is part of equality.** `Money` is a comparable struct, so `==`
  compares amount *and* currency: `NewMoney(1000, EUR) != NewMoney(1000, USD)`.
  The euros-vs-cents class of bug now fails at the type level.
- **A whitelist of currencies.** The *domain* decides which currencies the
  business supports; adding one is a one-line change that forces a moment of
  thought — better than silently accepting `"BTC"` or `"EURO"` from a client.

## Immutability changes arithmetic

There is no `SetCents`. Operations return a *new* value and make you decide the
hard question up front — e.g. an `Add` returns an error for mismatched currencies
rather than silently converting, because conversion is an explicit domain
operation with a rate and a timestamp. The value object makes you answer once, in
one place.

## Revalidate at every boundary — especially JSON

A value object is only as good as its boundaries. `encoding/json` normally writes
straight into struct fields via reflection, bypassing your constructor. With
unexported fields it can't — so implement the interfaces and route
deserialization *back through* the constructor:

```go
// UnmarshalJSON goes through NewMoney so a Money can never be deserialized
// into an invalid state.
func (m *Money) UnmarshalJSON(data []byte) error {
	var raw moneyJSON
	if err := json.Unmarshal(data, &raw); err != nil {
		return err
	}
	money, err := NewMoney(raw.Cents, raw.Currency)
	if err != nil {
		return err
	}
	*m = money
	return nil
}
```

Why bother when the value was valid at serialization time? Because JSON doesn't
only come from you — it comes from an outbox row written by last year's code, a
message from another service, a hand-edited fixture. Revalidating on the way in
costs two comparisons and closes the whole category. The same logic applies at
the DB boundary: reconstruction from a row routes through `NewMoney`
([`repositories.md`](repositories.md)).

At the REST edge the DTO doesn't expose the domain type at all — it carries
`price_cents` and `currency` as explicit primitives. The name `price_cents` does
real work: no client developer wonders whether to send `19.99` or `1999`.

## Sharp edges to know

- **`String()` assumes two decimal places** — correct for EUR/USD, wrong for JPY
  (0) or KWD (3). The whitelist keeps the assumption safe by construction; going
  properly multi-currency means per-currency exponents from ISO 4217, and "cents"
  becomes the more accurate "minor units."
- **Division rounds.** 100 cents split three ways is 33+33+34. You need an
  allocation strategy (largest-remainder), and if you allocate, *persist the
  allocation* — a refund must mirror the original split, not recompute it.
- **Parse currency-aware at ingestion.** A bank API sending `"5000"` JPY means
  5000 yen, not 50.00 of anything.

## Beyond Money

The pattern generalizes to anything defined by its value and burdened with rules:
email addresses, percentages, date ranges, country codes, quantities-with-units.
The test for "should this be a value object?": *do I keep validating this same
shape in multiple places?* If yes, give it a constructor and a type and delete
the scattered checks.
