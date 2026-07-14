---
name: pagination
description: Conventions for paginating a collection or list endpoint in a REST API following the adapted Zalando guidelines. Use when paginating a collection/list endpoint, choosing cursor vs offset pagination, or designing a paged list response and its query parameters.
---

# Pagination

Apply these rules when designing a collection / list endpoint that returns
multiple items. Each rule is distilled inline so this skill is usable on its
own; cite the bundled chapter only for full detail.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](../../ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.

## Rules

- [ ] **Paginate every collection** that can grow beyond a few hundred entries
  (#159). This protects the service from overload and supports client-side
  iteration/batch processing.
- [ ] **Prefer cursor-based over offset-based pagination** (#160). Choose by
  fit:
  - **Cursor-based** — default. Best for large/growing data sets, sharded or
    NoSQL stores, and `next`/`prev` navigation. The `cursor` is an **opaque**
    token clients must never inspect or construct; it encodes the page
    position, direction, and applied filters.
  - **Offset-based** — acceptable when clients genuinely need to *jump to page
    N* and the data set is small/stable. Downsides: duplicates or skipped rows
    when the underlying data changes between requests, and poor performance on
    large data sets.
- [ ] **The response body MUST be a JSON object**, never a bare array (#110).
  Wrap the page in an object with an **`items`** array plus pagination metadata
  (#161/#248). This is why #110 forbids top-level arrays: a bare `[...]` leaves
  no room for the metadata below.
- [ ] Use the **conventional query parameters** (#137), always **snake_case**:
  `cursor` and `limit` for cursor-based; `limit` and `offset` for offset-based;
  `q`/`sort`/`fields` for search/sort/field-selection.
- [ ] **Avoid a total result count** (#254). Computing it usually forces a full
  index scan and gets costlier as data grows; once clients depend on it,
  removing it is a breaking change. If truly required, expose it via the
  `Prefer: return=total-count` header rather than always in the body.
- [ ] **Expose hypermedia page links** where you use hypertext controls (#161):
  provide `self`, `first`, `prev`, `next`, `last` as **absolute URLs** sharing
  the request's base URL. Omit `prev` on the first page and `next` on the last.
  These may carry either full links or plain cursors (#248).

## Cursor-based example

Request:

```
GET /api/v1/orders?limit=50&cursor=b3JkXzk5MjM
```

Response (`200`):

```json
{
  "self":  "https://api.example.com/api/v1/orders?limit=50&cursor=b3JkXzk5MjM",
  "next":  "https://api.example.com/api/v1/orders?limit=50&cursor=b3JkXzEwNDcz",
  "prev":  "https://api.example.com/api/v1/orders?limit=50&cursor=b3JkXzk0MTA",
  "items": [
    { "id": "ord_9923", "status": "SHIPPED" },
    { "id": "ord_9924", "status": "PENDING" }
  ]
}
```

## Offset-based example

Request:

```
GET /api/v1/orders?limit=50&offset=100
```

Response (`200`):

```json
{
  "self":  "https://api.example.com/api/v1/orders?limit=50&offset=100",
  "next":  "https://api.example.com/api/v1/orders?limit=50&offset=150",
  "prev":  "https://api.example.com/api/v1/orders?limit=50&offset=50",
  "items": [
    { "id": "ord_1101", "status": "DELIVERED" },
    { "id": "ord_1102", "status": "SHIPPED" }
  ]
}
```

Note: object top level with an `items` array (#110/#161), opaque `cursor` /
numeric `offset` with `limit` (#137/#160), no `total_count` (#254), pluralized
`items` and `UPPER_SNAKE_CASE` enum `status` per the JSON conventions.

## Reference

For full detail, see the guidelines bundled with this plugin (the `reference/`
directory at the plugin root, e.g. `../../reference/<file>`):

- `reference/pagination.md` — #159, #160 (cursor vs offset), #161 (page links),
  #248 (page object), #254 (avoid total count)
- `reference/json-guidelines.md` — #110 (top-level object, never a bare array)
