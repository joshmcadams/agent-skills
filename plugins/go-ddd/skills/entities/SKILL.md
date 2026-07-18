---
name: entities
description: How to design domain entities in a Go DDD service so they guard their own invariants — NewX constructors instead of struct literals, invariants in a validate() method, the ValidatedX wrapper that makes "was validated" a compile-time fact, mutation through methods, and sentinel validation errors. Use when creating or reviewing a domain entity, deciding where a business rule/invariant lives, or fixing an anemic model with public fields and scattered validation.
type: knowledge
---

# Entities that guard themselves

Design guidance for **entities** — domain objects with identity and a life cycle
(a `Product`, a `Seller`) — so their invariants can't be bypassed. The enemy is
the anemic model: a struct of public fields, no constructor, rules smeared across
handlers as `if` statements that drift apart.

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This is a **knowledge** skill — advise; don't modify files unless asked. To add an
entity end-to-end (with repository, persistence, use cases, endpoints), use
**add-aggregate**.

## The four rules

1. **Constructors, not struct literals.** Every entity is built through a `NewX`
   that establishes every invariant at birth — assigns identity (`uuid.Must(
   uuid.NewV7())`, time-ordered), sets timestamps, records any creation event. A
   composite literal of a domain entity outside the `entities` package (and its
   tests) is a review flag.
2. **Invariants live in `validate()`** — one method that is the executable answer
   to "what are the rules for a product?" Each guard wraps a sentinel with `%w`:
   `return fmt.Errorf("%w: name must not be empty", ErrValidation)`.
3. **The `ValidatedX` wrapper makes validation a type.** An unexported
   `isValidated` field means the only way to get a `ValidatedProduct` outside the
   package is `NewValidatedProduct`, which runs `validate()`. Repository write
   methods take `*ValidatedX`, so "went through validation" is enforced by the
   compiler at the persistence boundary, not by a review comment.
4. **Mutation goes through methods that re-validate.** `func (p *Product)
   UpdatePrice(price Money) error { p.Price = price; p.UpdatedAt = time.Now();
   return p.validate() }`. No setter skips the check.

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

## Where a rule goes

- Involves only this entity's own fields → in its `validate()`.
- Spans another aggregate → enforce it at the boundary through the type system
  (constructors take `ValidatedX` refs) or as an application use case; it's usually
  *not* a hard per-commit invariant. See **aggregates**.
- Is about request shape, not business truth (is `price_cents` a number?) → that's
  edge validation in the interface layer, kept *in addition to* domain validation.
  Domain validation runs no matter the caller (HTTP, a consumer, a backfill).

## Honest limits (state these when advising)

Fields are exported because infrastructure must read them to persist and Go has no
`friend` access, so a determined colleague *can* assign a field directly and skip
validation. The pattern's claim is modest and enough: the convenient path and the
reviewed path are the safe one, and the repository boundary demands `ValidatedX`.
What keeps the model non-anemic is behavior on the entity plus re-validation on
every mutation — not field privacy.

## Applying these rules

- **Designing:** write `validate()` first (it's the design), then the constructor,
  then the `ValidatedX` wrapper, then mutation methods. Add sentinels to
  `errors.go`. Write pure domain tests asserting `assert.ErrorIs(t, err,
  ErrValidation)` per invariant — see **testing**.
- **Reviewing:** flag struct literals of entities outside `entities`, any write
  path that accepts a plain entity instead of `ValidatedX`, mutation that skips
  `validate()`, and validation branching on error strings instead of sentinels.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/entities.md` — the full treatment with code
- `reference/value-objects.md` — the same "invalid states unconstructible" idea for values
- `reference/aggregates.md` — which rules are per-commit invariants vs cross-aggregate
- `reference/conventions.md` — sentinel errors, constructors, defaults-in-domain
