---
name: add-collection
description: Add a new collection resource (list + create, plus the item sub-path) to a REST API following the adapted Zalando REST API guidelines. Use when the user asks to add a new collection to a REST API, add a new list/collection endpoint, or expose a new set of resources.
type: builder
argument-hint: "<[parent/]collection>"
---

# Add a Collection Resource

Add a new **collection** resource to the API: a plural endpoint that lists and
creates members, plus the member item sub-path with read/update/delete
operations. Wire the paths, operations, status codes, and JSON schemas so they
comply with the adapted Zalando REST API Guidelines bundled with this plugin.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](../../ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.

This skill **modifies the API artifact**. Produce the edited file(s) and a short summary of every change made.
## Arguments

`$ARGUMENTS` is a single path-like token — **never split it on spaces**; the `/`
inside it is meaningful. Parse it as `[parent/]collection`:

- **`orders`** → a top-level collection. Base path `/api/v1/orders`.
- **`customers/addresses`** → the `parent/` prefix (`customers`) names an
  **existing** parent collection; the new collection (`addresses`) hangs off a
  parent **item**. The nested base path is
  **`/api/v1/customers/{customer-id}/addresses`** — you MUST inject the parent's
  item identifier `{customer-id}` between the parent and the child. Do NOT
  produce `/api/v1/customers/addresses`.

If the argument is a single resource / singleton (no collection semantics), use
the **add-resource** skill instead.

## Step 1 — Detect the API artifact

1. Look first for an **OpenAPI document**: a `*.yaml`/`*.yml`/`*.json` file
   containing `openapi:` or `swagger:` (common names/locations: `openapi.yaml`,
   `api.yaml`, under `api/`, `spec/`, `docs/`). This is the primary target.
2. If none exists, degrade gracefully to **code-first** routes (Spring
   `@RequestMapping`/`@GetMapping`, FastAPI/Flask decorators, Express/Koa
   routers, ASP.NET controllers, etc.) and apply the change in that idiom.
3. **Tell the user which artifact you detected** and are editing. If a `parent/`
   prefix was given, confirm the parent collection already exists in the
   artifact (find its item path and identifier); if it does not, say so and stop.

## Step 2 — Name the collection

- **Pluralize** the collection name (#134) — it is a set of members, e.g.
  `orders`, `addresses`, `shipment-orders`.
- **kebab-case** every path segment, matching `^[a-z][a-z\-0-9]*$` (#129), e.g.
  `sales-orders` — never `salesOrders` or `sales_orders`.
- Use **domain-specific** names (#142): `sales-orders` beats `orders`; `items`
  alone is too generic.
- Keep URLs **noun-based / verb-free** (#141, #138) — no `/create-order`.

DO: `/api/v1/sales-orders`  ·  DON'T: `/api/v1/salesOrder`, `/api/v1/getOrders`

## Step 3 — Place it at the canonical path

- Serve under the **`/api` base path** followed by the major version segment
  `v<major>` (#135, #114): canonical shape **`/api/v1/{collection}`**.
- Top-level example (`orders`): **`/api/v1/orders`**.
- Nested example (`customers/addresses`):
  **`/api/v1/customers/{customer-id}/addresses`** — each sub-path must itself be
  a valid resource reference (#143), so `/api/v1/customers/{customer-id}` and
  `/api/v1/customers` must already resolve.
- Keep nesting depth **≤ 3** sub-resource levels (#147); no trailing or
  duplicate slashes (#136).

## Step 4 — Define the collection operations

On the collection path (`/api/v1/{collection}`):

- **`GET` (list)** → `200`. The response body MUST be a **JSON object** wrapping
  the members in an `items` array with pagination metadata — never a bare array
  (#110). Support pagination (#159): include page links/cursors
  `self`/`first`/`prev`/`next`/`last` and the `items` array (#161, #248). Use
  conventional query params `cursor`/`limit` (or `offset`/`limit`), `sort`,
  `q`, `fields`, all **snake_case** (#137, #130). Example shape:

  ```json
  {
    "self":  "/api/v1/orders?cursor=abc",
    "next":  "/api/v1/orders?cursor=def",
    "items": [ { "id": "..." } ]
  }
  ```

- **`POST` (create)** → `201` with the created resource (including its
  server-assigned `id`) in the body, and the item URL in the **`Location`**
  response header. The client MUST NOT supply the resource identifier — the
  service assigns it. Use `202` only if creation is asynchronous. `POST` on a
  collection is not idempotent by default.

## Step 5 — Define the item sub-path

Add the member item path **`/api/v1/{collection}/{id}`** (for a nested
collection: `/api/v1/customers/{customer-id}/addresses/{address-id}`) with:

- **`GET`** → `200`, or `404` if the item does not exist.
- **`PUT`** (replace entire resource) → `200`/`204` on update, `201` if it
  created the resource; `404` if update-only and missing.
- **`PATCH`** (partial update) → `200`/`204`; declare a patch media type
  (`application/merge-patch+json` or `application/json-patch+json`).
- **`DELETE`** → `200`/`204` (or `202` if async); `404` if missing, `410` if
  already deleted.

Only use official status codes with their standard semantics (#243, #150).

## Step 6 — Schemas and errors

- Request and response payloads are **JSON objects** at the top level (#110),
  never arrays or maps. Design one schema shared for read and write where
  practical.
- Property names are **snake_case** matching `^[a-z_][a-z_0-9]*$` (#118); array
  properties are pluralized (#120); object names singular.
- Use standard field conventions: opaque string `id` (#174), date/times as
  `format: date-time` with an `_at`/`date`/`time` name (#235), money as the
  common Money object with a decimal-string amount (#173), enums as
  `UPPER_SNAKE_CASE` strings (#240).
- Every operation returns errors as **`application/problem+json`** (RFC 9457,
  #176). Reference a shared Problem schema; never leak stack traces (#177).

## Output

Produce:
1. The **edited artifact** (OpenAPI document, or the code-first equivalent) with
   the new collection path, item sub-path, operations, and schemas added.
2. A short **summary** listing every path and operation added (e.g.
   `GET /api/v1/orders`, `POST /api/v1/orders`, `GET /api/v1/orders/{id}`, …)
   with their status codes, and any assumptions made (detected artifact,
   confirmed parent for a nested collection).

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`../../reference/<file>.md`):

- `reference/urls.md` — #135 (`/api` base path), #134/#129/#142 (naming),
  #143/#147 (sub-resources & nesting), #137/#130 (query params)
- `reference/compatibility.md` — #114 (URL versioning), #110 (top-level objects)
- `reference/pagination.md` — #159/#160/#161/#248 (paginated list responses)
- `reference/http-requests.md` — #148/#149 (method semantics)
- `reference/http-status-codes-and-errors.md` — #150/#243 (status codes), #176 (problem+json)
- `reference/json-guidelines.md` — #118/#120/#174 (snake_case, arrays, common fields), #240 (enums)
