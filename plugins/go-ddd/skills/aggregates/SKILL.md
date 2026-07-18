---
name: aggregates
description: How to draw aggregate boundaries in a Go DDD service — cluster only what must be consistent in one transaction, make one entity the aggregate root, reference other aggregates by Id (never embed them), and keep aggregates small. Use when deciding whether something is its own aggregate or part of another, whether to store an Id or an embedded struct, where a cross-object business rule belongs, or when a load pulls too much or a rule spans independently-saved objects.
type: knowledge
---

# Aggregates and their boundaries

Design guidance for **aggregates** — clusters of objects that change together, each
with one entity as the **aggregate root** (the only entry point). The boundary
answers two questions: when I change this, what else must change *in the same
transaction*? And when I load it, how much comes with it?

> **These are opinions, not a standard.** Adapted from
> [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd); see the
> [`NOTICE`](${CLAUDE_PLUGIN_ROOT}/NOTICE). (If `${CLAUDE_PLUGIN_ROOT}` is
> undefined, the plugin root is two levels above this SKILL.md.)

This is a **knowledge** skill — advise; don't modify files unless asked. To add a
new aggregate end-to-end, use **add-aggregate**.

## The two rules

1. **Reference other aggregates by Id, never embed them.** `Product` stores
   `SellerId uuid.UUID`, not `Seller Seller`. Embedding causes torn ownership (two
   copies of a name, no source of truth), transactional creep (renaming a seller
   "modifies" every product), and loading bloat (can't fetch a product without its
   seller). An Id reference gives each aggregate one owner, one transaction, one
   loading story; when a use case needs both, the application layer loads both
   explicitly.
2. **Keep aggregates as small as their invariants allow.** Big aggregates serialize
   writes (every change locks the root), bloat loads, and turn migrations into
   archaeology. Most well-factored aggregates are one entity plus some value
   objects.

## Drawing the boundary

The criterion is **what must be immediately consistent**:

- An invariant that must hold at *every commit* → inside one aggregate.
- A rule that may lag a moment → spans aggregates, reconciled via events or edge
  checks. Choosing eventual consistency here is the design telling the truth about
  what the business requires, not a compromise.

A checklist:

1. List the invariants. Which must be true at *every* commit?
2. Group only what those force together; everything else is an Id reference.
3. Check write contention — will two users routinely modify this at once? Then it's
   too big.
4. Check the load — if the common read pulls megabytes, too big.

## Worked patterns

- *"Price must be positive"* → only the product; inside its `validate()`.
- *"A product must belong to a validated seller"* → spans both; enforced at
  creation through the type system (`NewProduct` demands a `ValidatedSeller`). This
  guarantees validity *at creation*, not forever — "seller later deleted" is a use
  case (delete/orphan/block products), in the application layer or reacting to a
  `SellerDeleted` event.
- *"Names unique per seller"* → don't grow `Seller` to contain all products just to
  fit the invariant; enforce with a DB unique index on `(seller_id, name)` — a
  deliberate, documented case of infrastructure enforcing a rule because it does so
  atomically and cheaply. Dogma loses to a unique index.

## Consequence

One repository per aggregate root, none for anything inside a boundary — you load
and store aggregates whole, by their root. That's what kills the N+1 / lazy-loading
class of problems. See **repositories**. A bounded context is the larger cousin:
when "product" starts meaning different things to different parts of the business,
you have two contexts — and Id references + domain events are exactly what make
extracting one later a directory move, not a rewrite.

## Applying these rules

- **Designing:** for each proposed cluster, name the per-commit invariant that
  forces it together; if you can't, split it and reference by Id.
- **Reviewing:** flag embedded aggregate structs (should be an Id), aggregates that
  load or lock far more than their invariants require, and cross-aggregate rules
  modeled as hard invariants when eventual consistency would do.

## Reference

`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`:

- `reference/aggregates.md` — the full treatment with the boundary checklist
- `reference/entities.md`, `reference/value-objects.md` — what lives inside a boundary
- `reference/repositories.md` — one repository per root; transactions as the consistency unit
