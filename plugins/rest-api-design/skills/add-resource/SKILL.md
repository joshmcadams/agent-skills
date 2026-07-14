---
name: add-resource
description: Add a single resource — a top-level resource/singleton or a sub-resource of an existing collection — to a REST API following the adapted Zalando REST API guidelines. Use when the user asks to add a new resource to a REST API, add a singleton, or add a single sub-resource under an existing collection item. This skill is for a SINGLE named resource or singleton; if the user wants a plural set of items, use add-collection instead.
type: builder
argument-hint: "<[collection/]resource>"
---

# Add a Single Resource

Add **one** resource to the API: either a top-level resource / singleton, or a
sub-resource that hangs off an existing collection's item. Wire the path,
methods, status codes, and JSON schemas so they comply with the adapted Zalando
REST API Guidelines bundled with this plugin.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

This skill **modifies the API artifact**. Produce the edited file(s) and a short summary of every change made.

## Arguments

`$ARGUMENTS` is a single path-like token — **never split it on spaces**; the `/`
inside it is meaningful. Parse it as `[collection/]resource`:

- **`shipment`** → a top-level resource or singleton at `/api/v1/...`.
- **`orders/invoice`** → the `collection/` prefix (`orders`) names an
  **existing** parent collection; the new resource (`invoice`) is a sub-resource
  of a parent **item**, so it hangs off the item and you inject the parent's
  identifier: **`/api/v1/orders/{order-id}/invoice`**.

**If the user actually wants a whole collection** (a plural set of members with
list + create semantics), stop and use the **add-collection** skill instead —
this skill adds a single resource only.

### Singleton vs. collection item

Decide which kind of single resource this is:

- **Resource singleton** — a resource that is genuinely one-of-a-kind in its
  context. Per #134 it is normally modeled as a **collection with cardinality
  1** (set `maxItems = minItems = 1` on the returned array) to keep the
  cardinality constraint explicit.
- **`self` pseudo-identifier** (#143 exception) — when the resource identifier
  comes from the **authorization** context (a token or session cookie) rather
  than a path segment, use `self` as the identifier, e.g. `/api/v1/employees/self`
  or `/api/v1/employees/self/personal-details`. This is distinct from a
  singleton: `self` means "the caller's own instance," not "the only instance."
- **Collection item** — a single member addressed by id under an existing
  collection, e.g. `/api/v1/orders/{order-id}`.

## Step 1 — Detect the API artifact

1. Look first for an **OpenAPI document**: a `*.yaml`/`*.yml`/`*.json` file
   containing `openapi:` or `swagger:` (common names/locations: `openapi.yaml`,
   `api.yaml`, under `api/`, `spec/`, `docs/`). This is the primary target.
2. If none exists, degrade gracefully to **code-first** routes (Spring
   `@RequestMapping`/`@GetMapping`, FastAPI/Flask decorators, Express/Koa
   routers, ASP.NET controllers, etc.) and apply the change in that idiom.
3. **Tell the user which artifact you detected** and are editing. If a
   `collection/` prefix was given, confirm the parent collection already exists
   in the artifact (find its item path and identifier); if it does not, say so
   and stop.

## Step 2 — Name the resource

- **kebab-case** every path segment, matching `^[a-z][a-z\-0-9]*$` (#129), e.g.
  `shipment-order` — never `shipmentOrder` or `shipment_order`.
- Use **domain-specific** names (#142): `sales-invoice` beats `invoice`; avoid
  overly generic segments.
- Keep URLs **noun-based / verb-free** (#141, #138) — model actions as
  resources (e.g. a lock is `PUT /article-locks/{id}`, not `/lock`).
- A resource *singleton* uses a **singular** segment; a collection uses a plural
  one (#134) — if you find yourself pluralizing, you probably want add-collection.

## Step 3 — Place it at the canonical path

- Serve under the **`/api` base path** followed by the major version segment
  `v<major>` (#135, #114): canonical shape **`/api/v1/{resource}`**.
- Top-level example (`shipment`): **`/api/v1/shipment`** (singleton) or
  **`/api/v1/shipments/{shipment-id}`** if it is really a collection member.
- Sub-resource example (`orders/invoice`): **`/api/v1/orders/{order-id}/invoice`**
  — reference sub-resources by name + identifier in path segments (#143). Each
  sub-path must itself be a valid reference: `/api/v1/orders/{order-id}` and
  `/api/v1/orders` must already resolve.
- Keep nesting depth **≤ 3** sub-resource levels (#147); no trailing or
  duplicate slashes (#136).

## Step 4 — Choose the methods

Pick methods appropriate to what the resource is:

- **Singleton / `self` / single sub-resource** (one instance, addressed
  directly without its own `{id}`):
  - **`GET`** → `200`, or `404` if not present.
  - **`PUT`** (replace) → `200`/`204` on update, `201` if it created the
    resource.
  - **`PATCH`** (partial update) → `200`/`204`; declare a patch media type
    (`application/merge-patch+json` or `application/json-patch+json`).
  - **`DELETE`** → `200`/`204` (or `202` async); `404` missing, `410` gone —
    only if the resource is deletable.
  - Do **not** add list/create semantics to a singleton.
- **Collection item** (`/{id}` under an existing collection): same `GET` /
  `PUT` / `PATCH` / `DELETE` set. If the caller also needs list + create on the
  parent collection, that belongs in **add-collection**. If the parent
  collection path already exists and merely lacks a `POST` (create) operation,
  add it with **add-endpoint** on the collection path — not with this skill.

Respect method properties (#149): `GET` safe & idempotent; `PUT`/`DELETE`
idempotent; `POST`/`PATCH` not idempotent by default. Use only official status
codes with their standard semantics (#243, #150).

## Step 5 — Schemas and errors

- Request and response payloads are **JSON objects** at the top level (#110),
  never arrays or maps. Design one schema shared for read and write where
  practical.
- Property names are **snake_case** matching `^[a-z_][a-z_0-9]*$` (#118); array
  properties are pluralized (#120); object names singular.
- Use standard field conventions: opaque string `id` (#174), date/times as
  `format: date-time` with an `_at`/`date`/`time` name (#235), money as the
  common Money object with a decimal-number amount (#173), enums as
  `UPPER_SNAKE_CASE` strings (#240).
- Every operation returns errors as **`application/problem+json`** (RFC 9457,
  #176). Reference a shared Problem schema; never leak stack traces (#177).

## Verify

1. Re-read the edited file; confirm it still parses (for YAML/JSON run a
   quick parse, e.g.
   `python -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <file>`).
2. Confirm every `$ref` added resolves within the document.
3. If a spec linter is available in the project (`spectral`, `redocly`,
   `swagger-cli`, `openapi-spec-validator`), run it and fix errors it reports
   on the touched path. Do not install tools unasked; if none is available,
   say so in the summary.
4. Spot-check the new path against this plugin's audit checklist: naming,
   status codes, problem+json, and (for a singleton) that no list/create
   semantics were accidentally added.

## Output

Produce:
1. The **edited artifact** (OpenAPI document, or the code-first equivalent) with
   the new resource path, its methods, and schemas added.
2. A short **summary** listing every path and operation added (e.g.
   `GET /api/v1/orders/{order-id}/invoice`, `PUT /api/v1/orders/{order-id}/invoice`, …)
   with their status codes, whether you modeled it as a singleton / `self` /
   sub-resource, and any assumptions made (detected artifact, confirmed parent).

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/urls.md` — #135 (`/api` base path), #134 (singleton as cardinality-1
  collection), #129/#142 (naming), #143/#147 (sub-resources, `self`, nesting)
- `reference/compatibility.md` — #114 (URL versioning), #110 (top-level objects)
- `reference/http-requests.md` — #148/#149 (method semantics & properties)
- `reference/http-status-codes-and-errors.md` — #150/#243 (status codes), #176 (problem+json)
- `reference/json-guidelines.md` — #118/#120/#174 (snake_case, arrays, common fields), #240 (enums)
