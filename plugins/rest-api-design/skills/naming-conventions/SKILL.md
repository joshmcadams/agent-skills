---
name: naming-conventions
description: Naming rules for REST APIs — path segments, resource/collection names, resource IDs, query parameters, and JSON property names. Use when naming a resource, path, field, or query parameter, or when deciding casing (kebab-case, snake_case) for any API identifier.
type: knowledge
---

# REST API Naming Conventions

Apply these naming rules whenever you design or review names in a REST API:
resource/collection names, path segments, resource identifiers, query
parameters, and JSON property names. The rules below are self-contained; the
`reference/` chapters are cited only for full detail.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

This is a **knowledge** skill — it produces guidance and reference material, not edits to an artifact. Advise; do not modify files unless asked.

## Path & resource names

- **kebab-case path segments** matching `^[a-z][a-z\-0-9]*$` (#129): first char a
  lowercase letter, then letters, dashes, or digits. Applies to concrete
  segments, not necessarily path-parameter names.
  - Good: `/api/v1/shipment-orders/{shipment-order-id}`
  - Bad: `/api/v1/shipmentOrders`, `/api/v1/Shipment_Orders`
- **Pluralize collection names** (#134). Model a singleton as a collection of
  cardinality 1. Exception: the pseudo-identifier `self`.
  - Good: `/api/v1/customers`, `/api/v1/customers/{id}`
  - Bad: `/api/v1/customer`
- **Use domain-specific, non-generic names** (#142). Prefer names that identify
  the business object.
  - Good: `sales-order-items`
  - Bad: `order-items` (weaker), `items` (too generic)
- **No verbs in URLs** (#141). Actions belong in HTTP methods; URLs are nouns.
  Model an action as a resource (e.g. a "letter box" collection).
  - Good: `POST /api/v1/cancellations`, `PUT /api/v1/article-locks/{article-id}`
  - Bad: `POST /api/v1/orders/{id}/cancel`, `/api/v1/getOrders`
- **URL-friendly resource identifiers** (#228): IDs must match
  `[a-zA-Z0-9:._\-/]*` (ASCII letters, digits, `_`, `-`, `:`, `.`, and — only for
  compound keys, #241 — `/`). IDs must never be empty.
  - Good: `de:1681e6b88ec1`, `5273gh3k525a`
  - Bad: an ID containing spaces, `#`, `?`, or non-ASCII characters

## Query parameter names

- **snake_case** query parameters, never camelCase (#130), matching the same
  form as JSON property names.
  - Good: `?sales_channel_id=sid-1`
  - Bad: `?salesChannelId=sid-1`
- **Stick to conventional names** for search/sort/filter/paginate (#137):
  - `q` — default free-text query (give an entity-specific alias, e.g. `sku`).
  - `sort` — comma-separated fields; prefix `+` ascending / `-` descending,
    e.g. `?sort=+id,-created_at`.
  - `fields` — subset of fields to return.
  - `embed` — sub-entities to expand inline.
  - `offset` — numeric offset of the first element on a page.
  - `cursor` — opaque page pointer (never constructed/inspected by clients).
  - `limit` — client-suggested page size.
  - Bad: `page`, `pageSize`, `orderBy`, `search` when the conventional name fits.

## JSON property & array names

- **snake_case property names**, never camelCase (#118), matching
  `^[a-z_][a-z_0-9]*$`.
  - Good: `customer_number`, `sales_order_number`, `billing_address`
  - Bad: `customerNumber`, `BillingAddress`
- **Pluralize array names; keep object names singular** (#120). Names should be
  consistent across the API.
  - Good: `"items": [...]`, `"addresses": [...]`, `"address": { ... }`
  - Bad: `"item": [...]`, `"addressList": [...]`

## Applying these rules

- **When designing:** choose names that satisfy every rule above, and place them
  in the canonical `/api/v1/{resources}` shape.
- **When reviewing:** flag each violation by rule number (e.g. camelCase path
  segment → #129; singular collection → #134; verb in URL → #141; camelCase JSON
  property → #118). Do not flag correct `/api/v1/...` paths.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/urls.md` — path/query naming: #129 (kebab-case), #134 (pluralize),
  #142 (domain-specific), #228 (URL-friendly IDs), #141 (verb-free), #130 & #137
  (query parameters), #135 (`/api` base path)
- `reference/json-guidelines.md` — #118 (snake_case properties), #120 (pluralize
  arrays)
- `reference/compatibility.md` — #114 (URL versioning), #115 (no media-type
  versioning)
