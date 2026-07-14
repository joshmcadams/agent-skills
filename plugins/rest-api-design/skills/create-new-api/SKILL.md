---
name: create-new-api
description: Scaffold a brand-new REST API specification (OpenAPI 3.x) that follows the adapted Zalando REST API guidelines (URL versioning, /api base path). Use when the user asks to create, bootstrap, scaffold, or start a new REST API from scratch, or "set up a new API spec".
type: builder
argument-hint: "[api-name]"
---

# Create a New REST API

Bootstrap a brand-new REST API specification from scratch that complies with the
adapted Zalando REST API Guidelines bundled with this plugin. Prefer **OpenAPI
3.x** (default to `openapi: 3.1.0`). Produce a minimal-but-complete starter
document with correct meta-information, `/api` base path, `v1` URL versioning, a
first resource collection, problem+json errors, and OAuth2/bearer security.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

This skill **modifies the API artifact**. Produce the edited file(s) and a short summary of every change made.

## Arguments

`$ARGUMENTS` (optional) is a human name/slug for the new API, e.g. `orders` or
`Parcel Service`. Use it to seed `info.title`, the file name, and the first
resource collection. If empty, ask the user for the API's purpose and its first
resource, or pick a sensible placeholder and say so.

## Step 1 — Detect the artifact / target location

1. Search the repo for an **existing OpenAPI document** (`*.yaml`/`*.yml`/`*.json`
   containing `openapi:` or `swagger:`; common names/locations `openapi.yaml`,
   `api.yaml`, under `api/`, `spec/`, `docs/`). If one already exists, do **not**
   silently overwrite it — tell the user and ask whether to add to it (consider
   the `add-collection`/`add-resource` skills instead) or create a new file.
2. If no spec exists, choose a conventional path such as `api/openapi.yaml` (or
   `openapi.yaml` at repo root) and state where you are creating it.
3. If the project is clearly code-first (Spring, FastAPI/Flask, Express, ASP.NET)
   with no spec, state the assumption and offer to either generate an OpenAPI
   document (recommended, spec-first) or scaffold routes in that framework's
   idiom. Always tell the user which artifact you produced.

## Step 2 — Meta-information (`info` block) (#218, #116, #215, #219)

Include **all** required meta-information — the starter must pass an audit:

- [ ] `info.title` — unique, functional, descriptive name (#218).
- [ ] `info.description` — a proper description of the API (#218).
- [ ] `info.version` — semantic version `MAJOR.MINOR.PATCH`; start at **`1.0.0`**
  (or `0.1.0` for early design) (#116). This is the spec document version and is
  **distinct** from the URL version segment.
- [ ] `info.contact` — `{name, url, email}` of the owning team (#218).
- [ ] `info.x-api-id` — a globally unique, **immutable** identifier; generate a
  fresh UUID (#215). Never copy an existing API without changing this.
- [ ] `info.x-audience` — exactly one of `component-internal`,
  `business-unit-internal`, `company-internal`, `external-partner`,
  `external-public` (#219). Ask the user if unknown; default to
  `company-internal` and say so.

## Step 3 — Servers and versioning (the deviations) (#135, #114)

- [ ] Declare a `servers` entry using the **`/api` base path**, e.g.
  `url: https://{host}/api` with a `host` variable (#135).
- [ ] Start the API at major version **`v1`**: every path key begins with the
  `/v1/` segment (#114). Combined with the server this yields
  `/api/v1/{resources}`.
- [ ] Only the **major** version appears in the URL; minor/patch changes never
  change the path (#114). Do **not** encode the version in the media type (#115).

## Step 4 — First resource(s) (#134, #129, #142, #143)

- [ ] Model the first resource as a **plural**, **kebab-case**, **domain-specific**
  collection, e.g. `/v1/shipment-orders` (#134 plural, #129 kebab-case
  `^[a-z][a-z\-0-9]*$`, #142 domain-specific — prefer `sales-orders` over
  `items`).
- [ ] Keep URLs **verb-free / noun-based** (#141, #138).
- [ ] Provide the collection + instance pair: `/v1/{resources}` and
  `/v1/{resources}/{id}` (#143). Nesting depth ≤ 3 (#147); ~4–8 resource
  types max per API (#146).
- [ ] Define standard operations: `GET` (list), `POST` (create), `GET` (read),
  `PUT`/`PATCH` (update), `DELETE`. Use conventional query params for lists
  (`q`, `sort`, `fields`, `limit`, `cursor`/`offset`), snake_case (#130, #137).

## Step 5 — JSON payloads (#110, #118, #120)

- [ ] Every response body is a **JSON object** at the top level, never a bare
  array — wrap lists as `{ "items": [...] }` so pagination can be added later
  (#110).
- [ ] Property names are **snake_case** `^[a-z_][a-z_0-9]*$` (#118); array
  property names are pluralized (#120); object names singular.
- [ ] `id` fields are opaque strings (#174); date-times use `format: date-time`
  and `_at`/date-ish names (#235); design a single schema for read+write using
  `readOnly`/`writeOnly` (#252).

## Step 6 — Errors: problem+json (#176, #151)

- [ ] Define a reusable **`Problem`** schema and return it as
  `application/problem+json` (#176, RFC 9457). You may inline the schema (fields
  `type`, `title`, `status`, `detail`, `instance`) or reference the bundled model
  at `${CLAUDE_PLUGIN_ROOT}/reference/models/problem-1.0.1.yaml` (copy it into the
  spec's `components/schemas` for a self-contained artifact).
- [ ] Add a `default` error response (using problem+json) to every operation, and
  specify operation-specific `4xx`/`5xx` responses where they carry meaning
  (#151). Use only standard status codes (#150, #243). Never expose stack traces
  (#177).

## Step 7 — Security (#104, #105, #225)

- [ ] Define a security scheme: `http`/`bearer` (JWT) or `oauth2` (#104).
- [ ] Apply a scope to **every** operation via `security` (#105). Name scopes
  `application-id.access-mode` (e.g. `orders-service.read`,
  `orders-service.write`) (#225). Use `uid` only to explicitly mark an
  intentionally unrestricted endpoint (#105).

## Starter document (worked example — obeys every rule above)

```yaml
openapi: 3.1.0
info:
  title: Orders API
  description: Manages customer sales orders.
  version: 1.0.0
  x-api-id: 4c8b2f10-2f1a-4b6e-9d7c-1a2b3c4d5e6f   # fresh UUID; immutable
  x-audience: company-internal
  contact:
    name: Orders Team
    url: https://example.com/teams/orders
    email: orders@example.com
servers:
  - url: https://{host}/api           # /api base path (#135)
    variables:
      host:
        default: api.example.com
security:
  - BearerAuth: [ orders-service.read ]
paths:
  /v1/sales-orders:                    # -> /api/v1/sales-orders (#114, #134, #129, #142)
    get:
      summary: List sales orders
      operationId: listSalesOrders
      security:
        - BearerAuth: [ orders-service.read ]
      parameters:
        - { name: limit,  in: query, schema: { type: integer, minimum: 1, maximum: 100 } }
        - { name: cursor, in: query, schema: { type: string } }
      responses:
        '200':
          description: A page of sales orders.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/SalesOrderPage' }
        default:
          description: Error - see status code and problem object.
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }
    post:
      summary: Create a sales order
      operationId: createSalesOrder
      security:
        - BearerAuth: [ orders-service.write ]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/SalesOrder' }
      responses:
        '201':
          description: Sales order created.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/SalesOrder' }
        default:
          description: Error - see status code and problem object.
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }
  /v1/sales-orders/{sales-order-id}:
    parameters:
      - { name: sales-order-id, in: path, required: true, schema: { type: string } }
    get:
      summary: Retrieve a sales order
      operationId: getSalesOrder
      security:
        - BearerAuth: [ orders-service.read ]
      responses:
        '200':
          description: The sales order.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/SalesOrder' }
        default:
          description: Error - see status code and problem object.
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }
components:
  securitySchemes:
    BearerAuth:                        # (#104)
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    SalesOrder:                        # top-level object, snake_case props (#110, #118)
      type: object
      properties:
        id:          { type: string, readOnly: true, description: Opaque order id. }
        customer_id: { type: string }
        status:
          type: string
          description: '[Extensible enum] order status.'
          examples: [ CREATED, PAID, SHIPPED, CANCELLED ]
        created_at:  { type: string, format: date-time, readOnly: true }
      required: [ customer_id ]
    SalesOrderPage:                    # pagination page object (#248, #161)
      type: object
      properties:
        self:     { type: string, format: uri }
        next:     { type: string, format: uri }
        prev:     { type: string, format: uri }
        items:
          type: array
          items: { $ref: '#/components/schemas/SalesOrder' }
      required: [ items ]
    Problem:                           # RFC 9457 (#176); mirrors problem-1.0.1.yaml
      type: object
      properties:
        type:     { type: string, format: uri-reference, default: 'about:blank' }
        title:    { type: string }
        status:   { type: integer, format: int32, minimum: 100, maximum: 599 }
        detail:   { type: string }
        instance: { type: string, format: uri-reference }
```

## Output / result

Produce the new OpenAPI file at the chosen location, plus a short summary:
- the artifact path you created and the OpenAPI version used;
- the base path + version convention applied (`/api` server + `/v1/` paths);
- the first resource(s), security scheme, and scopes defined;
- any assumptions (audience, generated `x-api-id`, resource name) the user
  should confirm, and suggested next steps (e.g. run the `audit` skill, add more
  collections with `add-collection`).

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/meta-information.md` — #218/#116/#215/#219 (`info`, version, `x-api-id`, audience)
- `reference/urls.md` — #135 (`/api` base path), #134/#129/#142/#143 (naming, structure)
- `reference/compatibility.md` — #114 (URL versioning), #115 (no media-type versioning), #110
- `reference/json-guidelines.md` — #118/#120/#174/#235/#252 (payload conventions)
- `reference/http-status-codes-and-errors.md` — #176/#151/#150 (problem+json, responses)
- `reference/security.md` — #104/#105/#225 (auth, scopes, naming)
- `reference/models/problem-1.0.1.yaml` — the bundled `Problem` schema
- `reference/index.md` — chapter map and adaptation summary
