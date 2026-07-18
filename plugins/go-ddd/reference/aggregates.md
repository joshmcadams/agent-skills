# Aggregates and their boundaries

**Aggregates** answer two questions every growing codebase faces:

1. When I change this object, what else must change *in the same transaction*?
2. When I load this object, how much of the object graph comes with it?

Get them wrong and you land in one of two failure modes: everything loads
everything (fetch a product, drag in its seller, the seller's other products, a
14-join query), or consistency rules quietly span independently-saved objects and
you get states the business considers impossible.

## The rule: reference other aggregates by Id

An aggregate is a cluster of objects that change together, with one entity as the
**aggregate root** — the only entry point. Everything inside the boundary is
loaded, changed, and saved together, in one transaction. Everything *outside* is
referenced by identity only.

`Seller` and `Product` are **separate aggregates**. `Product` stores a
`SellerId`, not an embedded `Seller`:

```go
type Product struct {
	Id       uuid.UUID
	Name     string
	Price    Money
	SellerId uuid.UUID // reference by Id — not: Seller Seller
	// ...
}
```

Embedding the full seller causes exactly the problems the rule predicts:

- **Torn ownership** — two copies of the seller's name in memory, no answer to
  which is true after an update.
- **Transactional creep** — renaming a seller technically modifies every product
  embedding it. Lock their rows? Bump their `UpdatedAt`? Nobody chose; the ORM
  chose.
- **Loading bloat** — you can't fetch a product without dragging its seller along.

With an Id reference, each aggregate has one owner, one transaction, one loading
story. When a use case needs both, the *application layer* loads both aggregates
explicitly — visible, intentional, reviewable.

## Boundaries come from invariants, not foreign keys

The mechanical rule ("reference by Id") is easy. The design question is *where to
draw the line*, and the criterion is: **what must be immediately consistent?**

- An invariant that must hold at *every commit* belongs inside one aggregate.
- A rule that may lag a moment can span aggregates and be reconciled through
  events or edge checks.

Worked examples:

- *"A product's price must be positive"* — involves only the product. Inside the
  `Product` aggregate, enforced in `validate()`.
- *"A product must belong to a seller that passed validation"* — spans both.
  Enforced at creation time through the type system: `NewProduct` demands a
  `ValidatedSeller`. Note the honesty: this guarantees the seller was valid *when
  the product was created*, not that it still exists a week later. Whether
  deleting a seller must handle their products is a **use case** (delete? orphan?
  block?), implemented in the application layer or as a reaction to a
  `SellerDeleted` event. Choosing eventual consistency between aggregates isn't a
  compromise — it's the design telling the truth about what the business requires.
- *"Product names unique per seller"* (hypothetical) — the classic trap. Don't
  make `Seller` a big aggregate containing all its products to fit the invariant
  inside one boundary; that serializes writes and bloats loads. Enforce it with a
  database unique index on `(seller_id, name)` — a deliberate, documented
  exception where infrastructure enforces a domain rule because it does so
  atomically and cheaply. Dogma loses to a unique index.

## Small aggregates win

The pull toward big aggregates is real — everything *feels* related. Experience
says the opposite: **aggregates should be as small as their invariants allow.**
Big aggregates serialize writes (every change locks the root), bloat loads, and
turn migrations into archaeology. Most well-factored aggregates are one entity
plus some value objects.

A checklist for drawing a boundary:

1. List the invariants. Which must be true at *every* commit?
2. Group only what those invariants force together. Everything else: Id reference.
3. Check write contention. Will two users routinely modify this aggregate at
   once? If yes, it's probably too big.
4. Check the load. If the common read pulls megabytes, same conclusion.

## Consequence for repositories

One repository per aggregate root — `ProductRepository`, `SellerRepository` — and
no repository for anything inside a boundary. You never load "a product's events"
or "half a seller"; you load aggregates, whole, by their root. This is what kills
the N+1 / lazy-loading class of problems. See [`repositories.md`](repositories.md).
