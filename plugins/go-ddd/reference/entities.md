# Entities that guard themselves

An **entity** is a domain object with identity and a life cycle. Two products
with identical attributes are still different products; identity, not attributes,
is what makes each one itself. At every point in its life certain things must
hold true — its **invariants** — and the entity's job is to make violating them
hard.

The anemic anti-pattern to avoid: a struct of public fields, no constructor, no
rules. Any code can build a product with an empty name and a negative price and
the compiler helps it. The rules then live only in scattered `if` statements
across handlers and jobs, drifting apart.

## Rule 1 — constructors, not struct literals

Every entity is built through a `NewX` constructor that establishes every
invariant at birth:

```go
func NewProduct(name string, price Money, seller ValidatedSeller) *Product {
	product := &Product{
		Id:        uuid.Must(uuid.NewV7()),
		CreatedAt: time.Now(),
		UpdatedAt: time.Now(),
		Name:      name,
		Price:     price,
		SellerId:  seller.Id,
	}
	product.recordEvent(events.NewProductCreated(
		product.Id, name, price.Cents(), string(price.Currency()), seller.Id))
	return product
}
```

Decided here, once, for the whole system:

1. **Identity is assigned by the domain** — a UUIDv7 (time-ordered, indexes
   well) generated in the constructor, never by the database or the caller. The
   entity has its Id before it ever touches Postgres.
2. **The price is a `Money`** — a value object that cannot exist invalid
   ([`value-objects.md`](value-objects.md)).
3. **The seller parameter is a `ValidatedSeller`, not a `Seller`** — you
   literally cannot call the constructor with an unvalidated one.

A struct literal for a domain entity outside the `entities` package (and its
tests) is a review flag — it bypasses the constructor.

## Rule 2 — the validated-entity pattern

The problem it solves: "has this struct been validated?" is otherwise a question
with no answer. Make validation a *type*. `isValidated` is unexported, so the
only way to get a `ValidatedProduct` outside the package is through the
constructor, which runs `validate()`:

```go
type ValidatedProduct struct {
	Product
	isValidated bool
}

func NewValidatedProduct(product *Product) (*ValidatedProduct, error) {
	if err := product.validate(); err != nil {
		return nil, err
	}
	return &ValidatedProduct{Product: *product, isValidated: true}, nil
}
```

`validate()` is where the invariants live — one function that is the executable
answer to "what are the rules for a product?":

```go
func (p *Product) validate() error {
	if p.Name == "" {
		return fmt.Errorf("%w: name must not be empty", ErrValidation)
	}
	if p.Price.Cents() == 0 {
		return fmt.Errorf("%w: price must be greater than 0", ErrValidation)
	}
	if p.SellerId == uuid.Nil {
		return fmt.Errorf("%w: seller id must not be empty", ErrValidation)
	}
	if p.CreatedAt.After(p.UpdatedAt) {
		return fmt.Errorf("%w: created_at must be before updated_at", ErrValidation)
	}
	return nil
}
```

The payoff is what downstream code can *demand*. The repository write methods
take the validated type:

```go
Create(ctx context.Context, product *entities.ValidatedProduct) (*entities.Product, error)
```

The signature **is** the guarantee: nothing reaches the database without passing
validation, and forgetting is a compile error, not a review miss.

## Rule 3 — mutation goes through methods

Entities change, and every change must re-establish invariants. Mutate through
methods that end in `validate()`:

```go
func (p *Product) UpdatePrice(price Money) error {
	p.Price = price
	p.UpdatedAt = time.Now()
	return p.validate()
}
```

## Rule 4 — validation errors are sentinels

Every check wraps `ErrValidation` with `%w`. The interface layer maps
`errors.Is(err, ErrValidation)` to a 400 without parsing message strings — the
domain speaks in errors, the edge translates them. Messages are documentation and
may be reworded; the sentinel is the contract. See
[`conventions.md`](conventions.md).

## Honest limits

Fields are exported because the infrastructure layer must read them to persist,
and Go has no `friend` access. A determined colleague *can* write `p.Price =
Money{}` directly and skip validation. The pattern's claim is modest and, in
practice, enough: the **convenient path and the reviewed path are the safe one**,
and the repository boundary demands the validated type. What keeps the model
non-anemic is not field privacy — it's that behavior lives on the entity and
every mutation path re-validates.

## Domain validation vs edge validation

Keep both; they answer different questions. The edge asks *is this request
well-formed?* (is `price_cents` a number, `seller_id` a UUID?) and produces fast,
friendly 400s. The domain asks *is this a valid product?* (is the price positive,
does the seller pass validation?) and runs no matter where the call came from —
HTTP today, a queue consumer tomorrow, a 2am backfill. Each rule gets its one
correct home.
