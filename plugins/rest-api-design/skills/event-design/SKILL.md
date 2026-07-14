---
name: event-design
description: Design event types for asynchronous, publish-subscribe event-driven interfaces — event categories, schemas, naming, ownership, compatibility mode, ordering, and partitioning. Use when the user asks to define an event, design an event type, publish a domain event or data-change event, or set up event schemas/compatibility for an event-driven architecture.
type: knowledge
---

# Event Type Design

Design an **Event Type** for a publish/subscribe, event-driven interface:
schema, category, naming, ownership, compatibility mode, ordering, and
partitioning. Events are a first-class service interface, held to the same
design bar as a REST API — treat this skill as the events counterpart to
`create-new-api`/`modify-schema`. This is a knowledge skill: it produces an
event type design, not edits to a REST API artifact (though the schema you
design may live in the same OpenAPI document, in `components/schemas`, or in
a separate event-type-registry file, depending on the project's tooling).

> **Adaptation:** URL versioning and `/api` base path apply to this plugin's
> REST APIs — see the [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md).
> Events are not URL-addressed, so the adaptation's two deviations don't apply
> directly to the event schema itself, but data-change events should still
> mirror the REST resource's snake_case/JSON conventions (#205).
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

This is a **knowledge** skill — it produces guidance and a designed event type
(schema + metadata), not edits to an artifact. Advise; do not modify files
unless asked.

## Events are a first-class interface (#194, #208, #195)

- Treat event design with the same "API first" rigor as a REST API: write the
  schema before implementing, and make it available for review before
  publishing (#195).
- Everything that applies to REST payloads applies to events too (#208):
  General Guidelines, Data Formats, JSON Guidelines, Hypermedia conventions —
  cross-reference the `json-conventions` and `naming-conventions` skills for
  the field-level rules (snake_case, pluralization, enums, dates, Money, etc.)
  and apply them to the event's custom payload exactly as you would a REST
  response body.

## Step 1 — Choose the event category (#198)

Every event type conforms to exactly one category:

- **General Event** — a general-purpose category for signaling steps in a
  business process (formerly called "Business Event"). The custom payload sits
  at the **top level** of the document, alongside a required `metadata` field.
- **Data Change Event** — signals a mutation (create/update/delete/snapshot)
  of a stored entity, for change-data-capture / replication use cases. The
  custom payload lives under a required `data` field, alongside required
  `metadata`, `data_type`, and `data_op` fields.

Use **General Event** (#201) when the event represents a step in a business
process, not a direct entity mutation. Use **Data Change Event** (#202) when
the event signals that an entity your service owns was created, updated,
deleted, or snapshotted — this is mandatory for change-data-capture use cases,
not optional.

### Data Change Event shape

```yaml
DataChangeEvent:
  required: [metadata, data_op, data_type, data]
  properties:
    metadata: { $ref: '#/definitions/EventMetadata' }
    data:
      type: object
      description: Custom payload, conforming to the schema declared in the event type.
    data_type:
      type: string
      example: 'sales_order.order'
    data_op:
      type: string
      enum: ['C', 'U', 'D', 'S']   # Create, Update, Delete, Snapshot
```

### General Event shape

```yaml
GeneralEvent:
  required: [metadata]
  properties:
    metadata: { $ref: '#/definitions/EventMetadata' }
    # custom payload fields go directly alongside metadata, at the top level
```

## Step 2 — Define the Event Type registration (#197, #196)

Register the event with:
- **`name`** — functional naming, `<functional-name>.<event-name>[.v<major>]`,
  matching `[a-z][a-z0-9-]*\.[a-z][a-z0-9-]*(\.[Vv][0-9.]+)?` (#213), e.g.
  `transactions-order.order-cancelled`,
  `customer-personal-data.email-changed.v2`. Prefer this over the deprecated
  `<application-id>.<event-name>` form, which is only allowed for internal-
  audience event types. Reuse the same entity naming as the corresponding
  REST API when one exists (#205).
- **`audience`** — same five values as REST API audience (#219):
  `component-internal`, `business-unit-internal`, `company-internal`,
  `external-partner`, `external-public`.
- **`owning_application`** — the producer application (and, by extension,
  team) responsible for the definition (#207). One clear owner even if
  multiple services can produce the same event kind.
- **`category`** — `data` or `general` (Step 1).
- **`compatibility_mode`** — see Step 4.
- **`schema`** — the OpenAPI Schema Object payload definition, semantically
  versioned (Step 5).

The event type as a whole conforms to the OpenAPI **Schema Object** dialect,
not full JSON Schema (#196):
- **Never use** these JSON Schema keywords in an event schema: `additionalItems`,
  `contains`, `patternProperties`, `dependencies`, `propertyNames`, `const`,
  `not`, `oneOf`.
- `additionalProperties` is redefined — see Step 3.
- `discriminator` is available for polymorphism instead of `oneOf`.
- `readOnly` is allowed but redundant (events are logically immutable).

## Step 3 — Design the schema to stay extensible (#210, #111)

- **Never** declare `additionalProperties: true` (a wildcard extension point)
  on a schema meant to evolve compatibly — instead, define new **optional**
  fields ahead of publishing them. This is the event-schema analog of REST's
  "never declare `additionalProperties: false`" rule (#111) — both rules exist
  to keep the schema open for compatible extension, just aimed at opposite
  failure modes (events: don't leave an unbounded wildcard; REST: don't close
  the object).
- Consumers **must** ignore fields they don't recognize — never treat the
  absence of `additionalProperties` as "the schema is closed."
- This constraint prevents *field redefinition* (a publisher later giving an
  already-emitted field a different type), the event equivalent of the
  breaking changes `modify-schema` blocks for REST fields.

## Step 4 — Choose the compatibility mode carefully (#245)

Pick exactly one, and treat it as a durable contract with every consumer:

| Mode | Meaning | Allowed schema changes |
|---|---|---|
| `compatible` | Every event ever published under any prior schema version still validates against the latest schema. | Only additive optional fields/definitions. Nothing else. |
| `forward` (default) | The *previous* schema can still read events produced under the *new* schema, assuming consumers follow the tolerant-reader principle (#108). | More permissive than `compatible`; still no breaking changes as seen by a tolerant reader. |
| `none` | Any modification accepted, including breaking ones. Undefined properties accepted unless declared. | Anything — use only when there is no compatibility guarantee to keep. |

Default to `forward` unless you have a specific reason to tighten to
`compatible` or loosen to `none`. Once chosen, changing the mode itself is a
significant decision — treat it like a breaking API change (coordinate with
consumers first).

## Step 5 — Version the schema semantically (#246)

- The `schema.version` field uses `MAJOR.MINOR.PATCH`, the event analog of
  `info.version` (#116).
- Under `compatible`/`forward` modes: only **PATCH** (description/title
  changes) or **MINOR** (new optional fields) revisions are allowed — **MAJOR**
  breaking changes are **not** allowed under these modes at all; if you need
  one, that's a signal to mint a **new event type name** with a `.v<n>` suffix
  instead (#213), not to bump the existing type's major version.
- Under `none`: PATCH, MINOR, or MAJOR changes are all permitted, since there
  is no compatibility guarantee to violate.
- Renaming/removing a field or adding a new **required** field is always
  MAJOR-level, i.e. never compatible under `compatible`/`forward`.

## Step 6 — Populate event metadata (#247, #211)

Every event (either category) carries a shared `EventMetadata` object:

- **`eid`** (required) — a UUID uniquely identifying this event instance
  within the event type's stream lifetime. **Reuse the same `eid` on retries**
  of the same logical event (e.g. via a deterministic UUID derived from the
  event's own fields, or persisting the id before the first publish attempt) —
  this is what lets consumers deduplicate (#211, #214).
- **`occurred_at`** (required) — RFC 3339 `date-time`, when the producer
  created the event object (may predate actual publication).
- **`event_type`**, **`received_at`**, **`version`**, **`partition`** — usually
  enriched by the broker/intermediary, not the producer; don't require the
  producer to set these.
- **`parent_eids`** (optional) — causal links to the event(s) that triggered
  this one, useful when a monotonic ordering value isn't available.
- **`flow_id`** (optional) — corresponds to the `X-Flow-ID` HTTP header, for
  correlating events with the request that triggered them.

## Step 7 — Provide ordering (#203, #242, #204)

- **General events**: ordering is a **SHOULD** — define `ordering_key_fields`
  (which field(s) carry the order) and, where relevant, `ordering_instance_ids`
  (which field(s) scope that order to a specific entity instance).
- **Data change events**: ordering is a **MUST** (#242) — change-data-capture
  consumers depend on it to keep replicas in sync. Exception: entities that
  are strictly append-only (never updated/deleted) may omit it.
- Prefer a monotonically increasing counter (a row-update version, a
  per-partition sequence) over a timestamp — clocks drift and can produce
  same-instant or out-of-order timestamps in distributed systems. If you must
  use a timestamp, use UTC and account for drift explicitly.
- For data change events, prefer the **`hash`** partition strategy, keyed on
  the entity identifier (not the `eid` or a timestamp) — this keeps all
  changes for one entity in the same partition, giving a locally ordered
  stream per entity even though cross-partition order isn't guaranteed. Only
  deviate from `hash` for a specific, stated reason.

## Step 8 — Design-quality checks before publishing

- [ ] **No sensitive data** unless the business genuinely needs it (#200) —
      personal data (email, address, etc.) in an event multiplies the surface
      area for access-control and compliance obligations.
- [ ] **Useful, not exploded** (#199) — model events around your service's
      actual resources/business processes; prefer a smaller number of
      abstract-enough event types over one per implementation detail.
- [ ] **Matches the REST representation where one exists** (#205) — reuse
      field names/shapes from the corresponding API resource unless the event
      is deliberately a different shape (e.g. a coarser aggregate, or a
      computed/matching result with no backing resource).
- [ ] **Idempotent, out-of-order-safe processing** (#212) — design so a
      consumer only interested in current state can safely skip
      older-than-last-processed events, typically by sending the entity id +
      a monotonic ordering value + the resulting state.
- [ ] **Consumers must be robust to duplicates** (#214) — document that
      dedup should key off `eid`; don't design a schema that makes
      deduplication impossible.

## Backward compatibility for events (#209)

Once published, every consumer must handle the event as-is — there's no
synchronous negotiation like a REST request/response. Apply #106's "don't
break consumers" discipline, with the event-specific compatible/incompatible
lists below (these differ from the REST rules in `api-compatibility` in a few
places — read them separately, don't assume REST's rules transfer exactly):

**Compatible (safe under `compatible`/`forward`):**
- Adding new optional fields.
- Reordering fields within an object, or reordering same-typed array values.
- No longer sending a value for an optional field (removing the *value*, not
  the field from the schema — the schema itself must still declare it, or a
  later re-add with a different type becomes possible).
- Removing an individual value from an enumeration.
- Adding new values to an **extensible** enum (#112/#108) — never to a closed
  `enum`.

**Incompatible (requires a new event type / major version, or `none` mode):**
- Removing a required field from the schema.
- Changing a field's default value.
- Changing a field's, object's, enum's, or array's type.
- Reordering values of *different* types within an array (a tuple).
- Adding a new optional field that redefines the meaning of an existing field
  (a co-occurrence constraint).
- Adding a value to a **closed** `enum` (use an extensible enum instead, #112).

When an incompatible change is unavoidable: align with **every** consumer
before publishing events under the new shape, and prefer minting a new event
type name (`<name>.v2`) over silently breaking the existing one — same
philosophy as `new-version` for REST APIs, applied to event streams instead of
URLs.

## Output / result

Produce: (1) the chosen event category and why, (2) the full Event Type
registration (`name`, `audience`, `owning_application`, `category`,
`compatibility_mode`, `schema` with semantic version), (3) the payload schema
itself (Data Change Event's `data` object, or General Event's top-level
fields) obeying the extensibility and JSON-Guidelines rules, (4) the ordering/
partitioning fields if applicable, and (5) a compatibility note — mode chosen
and what future changes it permits. Advise; don't create/modify files unless
the user asks you to write the schema into a specific artifact.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/events.md` — #194/#195/#197/#208 (events as first-class
  interface), #213/#207 (naming, ownership), #198/#201/#202 (categories),
  #247/#211 (metadata), #203/#242/#204 (ordering, partitioning), #196/#210
  (schema dialect, extensibility), #245/#246 (compatibility mode, semver),
  #200/#212/#214/#199/#205/#209 (design-quality and compatibility rules)
- `reference/compatibility.md` — #106 (don't break consumers), #108 (tolerant
  reader), #111/#112 (extensibility, extensible enums) — the REST-side
  analogs this skill's event rules parallel
- `reference/meta-information.md` — #116 (semantic versioning), #219 (audience)
- `reference/json-guidelines.md`, `reference/naming-conventions` skill —
  payload field conventions shared with REST responses
- `reference/index.md` — chapter map and adaptation summary
