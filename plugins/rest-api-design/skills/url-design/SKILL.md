---
name: url-design
description: How to design REST resource URLs — base path, URL versioning, resource and sub-resource path structure, nesting limits, compound keys, and conventional query parameters. Use when designing REST URLs, resource paths, or query parameters, or when structuring a new endpoint's path.
---

# REST URL Design

Apply these rules whenever you design or review the URL structure of a REST API:
the base path, the version segment, resource and sub-resource paths, nesting,
compound keys, and query parameters. The rules below are self-contained; the
`reference/` chapters are cited only for full detail.

> **Adaptation (authoritative — central to this skill):** This guideline set
> deviates from upstream Zalando in two ways that shape every URL. The major
> version is a **path segment** placed right after the base path (`/api/v1/...`),
> and resources are served under an **`/api` base path**. The canonical resource
> path is **`/api/v1/{resources}`**. Media-type/`Accept`-header versioning
> (`application/vnd...;version=2`) and root-served (no `/api`) paths are
> VIOLATIONS here; `/api/v1/...` is correct.

## Base path & versioning (the deviations — design these first)

- **`/api` base path** (#135): serve all resources under `/api`. This keeps API
  traffic routable via a gateway when the API shares a domain with a frontend.
  - Good: `/api/v1/customers`
  - Bad (violation): `/customers`, `/v1/customers` (root-served, no `/api`)
- **Major version as a path segment** immediately after `/api` (#114), of the
  form `v<major>` — `v` plus a positive integer, no leading zeros.
  - Good: `/api/v1/orders`, `/api/v2/orders`
  - Only the **major** version appears; minor, backward-compatible changes MUST
    NOT change the URL. A new major version is the tool of last resort — prefer
    compatible extensions (see the compatibility guidance).
- **No media-type versioning** (#115): do NOT encode the version in the
  `Accept`/`Content-Type` header. Keep the media type plain `application/json`.
  - Bad (violation): `Accept: application/x.zalando.cart+json;version=2`

## Resource & sub-resource structure

- **Identify resources and sub-resources via path segments** (#143):
  `/parents/{parent-id}/children/{child-id}`. Every sub-path must itself be a
  valid resource reference — if `/api/v1/partners/{id}/addresses/{addr}` is
  valid, so are `.../addresses`, `/partners/{id}`, and `/partners`.
  - Exception: use `self` as a pseudo-identifier when the resource is identified
    by the auth token, e.g. `/api/v1/employees/self`.
- **Pluralize collection names** (#134) and use **kebab-case** segments matching
  `^[a-z][a-z\-0-9]*$` (#129).
  - Good: `/api/v1/shipment-orders/{shipment-order-id}`
- **Normalized paths** (#136): no trailing slash, no empty/duplicate segments.
  Path variables must never be empty strings.
  - Bad: `/api/v1/customers/`, `/api/v1/customers//addresses`
- **Nested vs. non-nested URLs** (#145): nest a sub-resource only if it cannot
  exist without its parent. If a resource has a globally unique ID and may be
  accessed directly, expose it as a top-level resource too.
  - Nested: `/api/v1/shopping-carts/de/1681e6b88ec1/items/1`
  - Top-level: `/api/v1/sales-orders/5273gh3k525a`
- **Limit resource types** (#146): follow functional segmentation; well-defined
  APIs expose ~4–8 resource types. Split larger domains into separate APIs.
- **Limit sub-resource nesting to ≤ 3 levels** (#147): deeper nesting increases
  complexity and path length.
- **Compound keys** (#241): a resource identified by multiple other IDs may use
  the compound key with slashes in the path instead of an opaque technical ID.
  Apply the compound-key abstraction consistently; pass components as independent
  query params only on search/create.
  - Good: `/api/v1/shopping-carts/{country}/{session-id}/items/{item-id}`
  - Good: `/api/v1/article-size-advices/{sku}/{sales-channel}`

## Query parameters

- **snake_case** query parameters, never camelCase (#130).
- **Conventional names** (#137): `q`, `sort` (`+`/`-` prefixes for direction),
  `fields`, `embed`, `offset`, `cursor`, `limit`.
  - Good: `/api/v1/sales-orders?sort=-created_at&limit=50&fields=id,status`
  - Bad: `?orderBy=created&pageSize=50`

## Canonical example set (sample domain: a shop)

```http
/api/v1/customers
/api/v1/customers/{customer-id}
/api/v1/customers/{customer-id}/addresses
/api/v1/customers/{customer-id}/addresses/{address-id}
/api/v1/sales-orders
/api/v1/sales-orders/{order-id}
/api/v1/sales-orders/{order-id}/line-items
/api/v1/sales-orders/{order-id}/line-items/{line-item-id}
/api/v1/shopping-carts/{country}/{session-id}          # compound key (#241)
/api/v1/sales-orders?sort=-created_at&limit=50          # conventional query (#137)
```

## Applying these rules

- **When designing:** build every path as `/api/v1/{resources}` (#135, #114),
  keep segments plural + kebab-case, nest ≤ 3 levels, and use conventional
  snake_case query params.
- **When reviewing:** flag violations by rule number — root-served / no-`/api`
  path → #135; missing version segment → #114; media-type versioning → #115;
  camelCase or singular segment → #129/#134; trailing/duplicate slash → #136;
  nesting > 3 → #147. Do NOT flag correct `/api/v1/...` paths.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`../../reference/<file>.md`):

- `reference/urls.md` — #135 (`/api` base path), #143 (sub-resources), #134
  (pluralize), #129 (kebab-case), #136 (normalized paths), #145 (nested URLs),
  #146 (resource-type limit), #147 (nesting depth), #241 (compound keys), #130 &
  #137 (query parameters)
- `reference/compatibility.md` — #114 (URL versioning), #115 (no media-type
  versioning), #113 (avoid versioning)
