---
name: add-endpoint
description: Add a single new operation (an HTTP method on a resource path) to an existing REST API, following the adapted Zalando guidelines (URL versioning, /api base path). Use when the user asks to add a new endpoint or operation to an existing REST API resource.
argument-hint: "<METHOD path>"
---

# Add an Endpoint to an Existing REST API

Add ONE operation — a single HTTP method on a resource path — to an existing API,
keeping it consistent with the surrounding design and the adapted Zalando REST
API Guidelines bundled with this plugin. Detect the API artifact, verify the
path and method, define request/response bodies, status codes, and headers, then
edit the artifact and summarize the change.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](../../ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.

## Arguments

`$ARGUMENTS` (optional) names the operation to add as a `METHOD path` pair, e.g.
`POST /orders/{id}/cancellations` or `GET /orders/{id}/line-items`. Treat the
**whole string** as one argument — do not split it into indexed args; the `/`
inside the path is meaningful. The first token is the HTTP method; the rest is
the resource path (relative to `/api/v{major}` — add the base path/version if the
user omitted it). If `$ARGUMENTS` is empty, ask the user which method and path to
add, or infer it from their request.

## Step 1 — Detect the API artifact

1. Look first for an **OpenAPI document**: a `*.yaml`/`*.yml`/`*.json` file
   containing `openapi:` or `swagger:` (common names/locations: `openapi.yaml`,
   `api.yaml`, `api/`, `spec/`, `docs/`).
2. If none exists, degrade gracefully to **code-first** routes: Spring
   (`@RequestMapping`/`@GetMapping`/`@PostMapping`), FastAPI/Flask decorators,
   Express/Koa routers, ASP.NET controllers, etc. State the assumption you are
   making and apply the change in that idiom.
3. Tell the user **which artifact you detected** before editing it.

## Step 2 — Verify the path fits the existing structure

- [ ] Path sits under the **`/api` base path** and a **major version segment**
  `v<n>`, i.e. `/api/v1/…` (#135, #114). If the surrounding API deviates (root-
  served, media-type versioned), FLAG it and follow the canonical scheme for the
  new operation; note the deviation to the user.
- [ ] New collection segments are **plural** (#134) and **domain-specific**
  (#142) — e.g. `sales-orders`, not `items`.
- [ ] Path segments are **kebab-case** `^[a-z][a-z\-0-9]*$` (#129); no
  trailing/duplicate slashes (#136).
- [ ] Sub-resources are addressed as path segments
  `/parents/{id}/children/{id}` (#143); keep nesting depth ≤ 3 (#147) and reuse
  existing resource types rather than inventing new ones (#146).
- [ ] Reuse the resource/id naming already present in the spec — match siblings.

## Step 3 — Pick the correct METHOD by semantics (#148, #149)

Choose the method from what the operation *means*, not from convenience:

- **GET** — read a single resource or a collection. **Safe** (no side effects)
  and idempotent. Never give GET a request body; put filters in query params.
- **POST** — create a resource in a collection, or perform an action modeled as a
  sub-resource. **Not** safe, **not** idempotent by default (#231/#229 to make it
  so). Use for creation and for anything the other methods cannot express.
- **PUT** — replace an **entire** resource (create-or-replace). **Idempotent.**
  Use only when the client controls the identifier; otherwise prefer POST for
  creation and reserve PUT for full updates.
- **PATCH** — **partial** update. Not idempotent by default. Use a documented
  patch media type: `application/merge-patch+json` (RFC 7396) or
  `application/json-patch+json` (RFC 6902). Pick exactly one per endpoint.
- **DELETE** — remove a resource. **Idempotent.**
- **HEAD** — like GET but headers only (safe, idempotent); pairs well with `ETag`.

Verb-free URLs: model actions as **sub-resources / nouns**, never verbs in the
path (#141, #138). E.g. to cancel an order, do NOT `POST /orders/{id}/cancel`;
instead `POST /api/v1/orders/{id}/cancellations` (post a message to the
"cancellations" letter box). To lock an article: `PUT
/api/v1/article-locks/{article-id}`.

## Step 4 — Define request & response bodies as JSON objects

- [ ] Top-level request and response payloads are **JSON objects**, never bare
  arrays or maps (#110/#167) — even collections wrap items in an object.
- [ ] Property names are **snake_case** `^[a-z_][a-z_0-9]*$` (#118); arrays are
  pluralized, objects singular (#120).
- [ ] Media type is plain `application/json` (#172) — no custom
  `application/x…+json` or version parameters.
- [ ] Booleans are never `null` (#122); `null` and absent mean the same (#123).
- [ ] Reuse common field names/objects where they apply: `id` (opaque string),
  `xyz_id`, money as the `{amount, currency}` object (#173), dates as RFC 3339
  `date`/`date-time` with `_at`-style names (#235).
- [ ] For creation, the client MUST NOT send the resource id — the server assigns
  it (#231 warning).

## Step 5 — If it's a collection GET, apply pagination

- [ ] Any endpoint returning a list MUST **paginate** (#159). Prefer **cursor**-
  based; offset-based is acceptable (#160).
- [ ] Use conventional query params only: `q`, `sort`, `fields`, `limit`,
  `cursor`, `offset`, `embed` (#137); query params are **snake_case** (#130).
- [ ] For multi-value query/header params (e.g. `sort`, filter lists), define the
  collection format explicitly — comma-separated or repeated (#154).
- [ ] Wrap results in a page object: `items` array plus `self`/`first`/`prev`/
  `next`/`last` links or cursors (#248/#161). Avoid a total count (#254).

## Step 6 — Choose and document status codes (#150, #151, #243)

Use **only** standard, well-understood codes with their RFC semantics; document
success and service-specific error responses per operation. By method:

- **GET/HEAD** — `200`; `404` when the resource/collection is missing.
- **POST (create)** — `201` (with `Location` to the new resource) or `200`/`204`;
  `202` if processed asynchronously (#253); `409` on a conflicting duplicate.
- **POST (batch/bulk)** — `207` multi-status with per-item results (#152).
- **PUT** — `200`/`204` on update, `201` on create, `202` async.
- **PATCH** — `200`/`204`, `202` async.
- **DELETE** — `200`/`204`, `202` async; `404`/`410` when already gone.
- **Any** — `400` for bad/invalid input (use `400`, **not** `422`, for business/
  semantic validation failures too); `401`, `403`, `404`, `406`, `429`
  (rate-limited, with `Retry-After`), `500`/`503` as applicable.
- Do NOT invent non-standard codes and do NOT use redirection codes (#251).
- Errors are returned as **`application/problem+json`** (RFC 9457, #176).

## Step 7 — Add the relevant headers

- [ ] On successful create, return the new resource URL in the **`Location`**
  header (#133); prefer `Location` over `Content-Location` (#180).
- [ ] For optimistic locking on `PUT`/`PATCH`/`DELETE`, support **`ETag`** with
  **`If-Match`**; a failed precondition returns `412` (#182). Use `If-None-Match:
  *` to guard creation.
- [ ] Header names are `Kebab-Case` (#132); only standard or the sanctioned
  proprietary `X-` headers (#183); do not encode business semantics in headers.

## Output

1. State the detected artifact and the final operation (`METHOD /api/v1/…`).
2. **Edit the artifact** to add the operation: path + method, parameters,
   request body schema, response schemas with status codes and problem+json
   errors, and headers. Match the file's existing style and reuse components.
3. Provide a short **summary**: the method + path, chosen status codes, request/
   response shape, headers added, and any **deviations you flagged or corrected**
   (e.g. added the missing `/api/v1` prefix, converted a verb path to a
   sub-resource). Do not make unrelated changes.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`../../reference/<file>.md`):

- `reference/urls.md` — #135 (`/api` base), #114 (URL versioning), #129/#134/#142
  naming, #141/#138 verb-free, #143/#146/#147 sub-resources, #130/#137 query params
- `reference/http-requests.md` — #148/#149 methods, #154, #229/#231, #253 async
- `reference/http-status-codes-and-errors.md` — #150/#151/#243 codes, #152, #153, #176
- `reference/json-guidelines.md` — #110/#167/#118/#120/#122/#123/#172/#173 payloads
- `reference/pagination.md` — #159/#160/#161/#248 pagination
- `reference/http-headers.md` — #133 `Location`, #182 `ETag`/`If-Match`, #132/#183
- `reference/index.md` — chapter map and adaptation summary
