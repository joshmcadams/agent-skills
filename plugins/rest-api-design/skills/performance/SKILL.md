---
name: performance
description: Reduce bandwidth and improve responsiveness in a REST API — gzip compression, partial responses via a fields query param, embedding sub-resources, and documenting cacheable endpoints with Cache-Control/Vary/ETag. Use when the user asks about API performance, response size, field filtering, embedding related resources, or whether/how to cache an endpoint.
type: knowledge
---

# REST API Performance & Caching

Apply these rules when a REST API has (or might grow) high payloads or is used
in high-traffic / low-bandwidth scenarios. This is a knowledge skill: it
produces performance and caching guidance, not edits to an artifact.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

This is a **knowledge** skill — it produces guidance and reference material, not edits to an artifact. Advise; do not modify files unless asked.

## Reducing bandwidth — the toolbox (#155)

Combine these techniques as needed; each is detailed below:
- `gzip` (or other) content compression.
- Partial responses via a `fields` query parameter.
- Embedding sub-resources via `embed` to cut round-trips.
- `ETag`/`If-Match`/`If-None-Match` conditional requests (see the
  `http-headers` skill for the mechanics).
- `Prefer: return=minimal` / `respond-async` to reduce processing/payload
  (see the `async-operations` skill).
- Pagination for large collections (see the `pagination` skill).
- Caching of rarely-changing master data (below).

## `gzip` compression (#156)

- Support `gzip` content encoding by default on both client and server, unless
  content is already compressed or the server lacks resources to compress.
- Negotiate via the standard headers: client sends `Accept-Encoding: gzip`
  (or a weighted list); server responds with `Content-Encoding: gzip` when it
  compresses. Always support uncompressed (`identity`) too, for testing.
- Declare both headers in the spec by referencing/copying the standard
  definitions rather than inventing custom ones (see
  `${CLAUDE_PLUGIN_ROOT}/reference/models/headers-1.0.0.yaml#/Accept-Encoding`
  and `#/Content-Encoding`).
- Most frameworks do **not** enable `gzip` out of the box — call this out as
  an implementation TODO (e.g. enable it explicitly in Spring Boot, or add a
  gzip middleware in Express/Gin), not just a spec concern.

## Partial responses via `fields` (#157)

- Let clients request a subset of a resource's fields with the conventional
  `fields` query parameter (naming per #137, the `url-design` skill), e.g.:
  ```
  GET /api/v1/users/123?fields=(name,friends(name))
  ```
  returns only `name` at the root and only `name` within each nested `friends`
  entry.
- The `fields` syntax follows this BNF grammar (reuse verbatim in the param
  description since OpenAPI can't express a conditional response shape):
  ```bnf
  <fields>            ::= [ <negation> ] <fields_struct>
  <fields_struct>     ::= "(" <field_items> ")"
  <field_items>       ::= <field> [ "," <field_items> ]
  <field>             ::= <field_name> | <fields_substruct>
  <fields_substruct>  ::= <field_name> <fields_struct>
  <field_name>        ::= <dash_letter_digit> [ <field_name> ]
  <dash_letter_digit> ::= <dash> | <letter> | <digit>
  ```
- Document the parameter with a line like "Endpoint supports filtering of
  return object fields" and a link/reference to this grammar, since OpenAPI
  itself cannot express the resulting variable schema.
- **Do not** give `fields` a default value — an implicit filter is
  counter-intuitive and clients won't expect it (principle of least
  astonishment).

## Embedding sub-resources via `embed` (#158)

- Let clients ask for related resources to be prefetched inline, avoiding
  extra round-trips, via the conventional `embed` query parameter (same BNF
  grammar as `fields`), e.g.:
  ```
  GET /api/v1/orders/123?embed=(items)
  ```
  ```json
  {
    "id": "123",
    "_embedded": {
      "items": [ { "position": 1, "sku": "1234-ABCD-7890", "price": { "amount": 71.99, "currency": "EUR" } } ]
    }
  }
  ```
- Whether embedding is a database join or a generic proxy-level expansion is
  an implementation detail — the contract is just the `_embedded` wrapper and
  the `embed` query syntax.
- Only offer `embed` where clients demonstrably need to avoid N+1 request
  patterns; don't embed everything by default.

## Cacheable endpoints (#227)

- **Default posture: not cacheable.** Servers and clients should assume
  `Cache-Control: no-store` unless a `Cache-Control` header says otherwise.
  There's nothing to document for this default — just make sure the framework
  actually sends it (many frameworks don't by default; verify explicitly,
  e.g. via Spring Security's cache-control config).
- **If you genuinely need caching** (e.g. a heavily used, rate-limited
  master-data service whose data rarely changes after creation), document the
  cacheable `GET`/`HEAD`/`POST` endpoints by declaring `Cache-Control`,
  `Vary`, and `ETag` in the response. Do **not** define an `Expires` header —
  it duplicates and can conflict with `Cache-Control`.
- Sensible defaults when you do enable caching:
  ```http
  Cache-Control: private, must-revalidate, max-age=300
  Vary: accept, accept-encoding
  ```
  Use `private` for endpoints behind standard OAuth authorization (per-caller
  data); tune `max-age` between seconds and a day depending on how often the
  data changes and how consistent clients need to be.
- Cache support patterns, from cheapest to most involved:
  - Every cacheable endpoint should support `ETag` + `If-Match`/`If-None-Match`
    (see the `http-headers` skill for the full mechanics).
  - For large single resources, support `HEAD` or a lightweight `GET` with
    `If-None-Match` to cheaply check for updates — return `304` (not `412`) on
    a failed `If-None-Match` check for `HEAD`/`GET`.
  - For small collections, put an `ETag` on the whole collection, checked via
    `HEAD`/`GET` + `If-None-Match`.
  - For medium collections, combine a collection-level `ETag` with pagination
    and an entity-tag-based "changes since" filter — note this pattern isn't
    supported by generic HTTP-layer caches, only application-level ones.
- Prefer attaching a (possibly distributed) cache at the service/gateway
  layer over relying on generic HTTP client/proxy caches — it's easier to
  reason about `Vary` correctly and supports the warm-up patterns above.
- Copy the `Cache-Control`, `Vary`, and `ETag` header definitions from
  `${CLAUDE_PLUGIN_ROOT}/reference/models/headers-1.0.0.yaml` into the spec's
  own `components/headers` and `$ref` them locally, same pattern as the
  `error-responses` and `deprecate` skills use for their shared headers.

## Output / result

Produce: (1) which bandwidth techniques apply to the endpoint(s) in question
and why, (2) the `fields`/`embed` parameter definitions if partial
responses/embedding are needed, and (3) a caching verdict — not cacheable
(default, nothing to add) or cacheable with the specific `Cache-Control`/
`Vary`/`ETag` values and cache-support pattern chosen. Advise; do not modify
files unless asked.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/performance.md` — #155 (technique overview), #156 (gzip), #157
  (`fields` partial responses), #158 (`embed`), #227 (cacheable endpoints)
- `reference/http-headers.md` — #182 (`ETag`/`If-Match`/`If-None-Match`
  mechanics, cross-reference the `http-headers` skill)
- `reference/urls.md` — #137 (conventional query parameter names)
- `reference/models/headers-1.0.0.yaml` — `Accept-Encoding`, `Content-Encoding`,
  `Cache-Control`, `Vary`, `ETag` header schemas to copy in
- `reference/index.md` — chapter map and adaptation summary
