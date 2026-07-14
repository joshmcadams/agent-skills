---
name: http-headers
description: How to choose and define HTTP headers in a REST API — casing, content negotiation, conditional requests, idempotency, proprietary headers, deprecation. Use when deciding which HTTP headers to use, adding custom/X-* headers, doing content negotiation, or setting up caching/ETag/optimistic locking.
---

# HTTP Headers

Guidance for selecting and defining HTTP headers in a REST API: standard headers
and casing, content negotiation, conditional requests / optimistic locking,
processing preferences, idempotency, the sanctioned proprietary headers, and
deprecation signaling. Any header a service supports is part of its contract, so
declare it explicitly in the OpenAPI `parameters`/`headers` of the operation.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](../../ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.

## Rules

1. **Prefer standardized headers with correct casing (#133, #132).** Use headers
   defined by non-obsolete RFCs and list the ones you support in the spec. Write
   header names in `Kebab-Case` with uppercase-separated words —
   `If-Modified-Since`, `Accept-Encoding`, `Content-Type` — even though HTTP
   treats field names case-insensitively. Keep common abbreviations uppercase
   (`ID`, `ETag`).

2. **Use `Content-*` headers correctly (#178).** `Content-*` (entity) headers
   describe the message body: `Content-Type`, `Content-Length`,
   `Content-Encoding`, `Content-Language`, `Content-Disposition`,
   `Content-Range`. Prefer the `Location` header over `Content-Location` for
   pointing at a created/redirected resource (#180).

3. **Content negotiation via `Content-Type`/`Accept` — but NOT for versioning
   (#244, #172, #115).** When a resource has multiple representations at the same
   URL (e.g. JSON, PDF, HTML), let clients choose with the standard `Accept`,
   `Accept-Language`, `Accept-Encoding` headers, using standard media types like
   `application/json` or `application/pdf`.
   **Deviation:** do NOT version the API through the media type. Custom versioned
   media types such as `application/vnd.x+json;version=2` or
   `application/x....+json;version=2`, and any `Accept`/`Content-Type`-based
   version negotiation, are a **violation** here — the major version lives in the
   URL path (`/api/v1/...`), and the media type stays plain `application/json`.

4. **Conditional requests & optimistic locking with `ETag` + `If-Match` /
   `If-None-Match` (#182).** Return an `ETag` (opaque quoted string: a hash,
   last-modified hash, or version id) on resources. To prevent lost updates,
   have clients send `If-Match: <etag>` on `PUT`/`POST`/`PATCH`; if it does not
   match the current version, respond **412 Precondition Failed**. Use
   `If-None-Match: *` to guard resource creation (fail with 412 if any matching
   entity already exists).

5. **`Prefer` / `Preference-Applied` for processing preferences (#181).** Support
   the RFC 7240 `Prefer` header (optional) to let clients request behaviors like
   `respond-async`, `return=minimal|representation`, `return=total-count`,
   `wait=<seconds>`, `handling=strict|lenient`. Prefer this over inventing `X-`
   headers for processing directives. Echo `Preference-Applied` when a preference
   was honored. Copy only the behaviors your endpoint actually supports.

6. **`Idempotency-Key` for safe retries (#230).** To make create/update
   operations idempotent across retries (timeouts, network outages), accept a
   client-generated unique key (UUIDv4 recommended) via the `Idempotency-Key`
   request header. The service caches the response against the key (e.g. 24h) and
   replays it instead of re-executing. This is a standard REST concern, not a
   proprietary header — do not rename it or `X-`-prefix it.

7. **Avoid proprietary headers; define and propagate them when necessary (#183, #184).**
   As a general rule, express business intent via path, query, method, and body —
   not proprietary headers. When domain-specific context must be passed
   end-to-end (e.g. a correlation ID for tracing, or tenant/device context from a
   gateway), you may define proprietary headers under these conventions:
   - Prefix with `X-` and use dash-case (`X-Example-Header`). Note: RFC 6648
     deprecates `X-`, but it remains the widely recognized convention.
   - Name them consistently and scope them to avoid collisions (e.g.
     `X-MyOrg-...`). Document their purpose, allowed values, and propagation
     expectations in the API spec.
   - Mark them as end-to-end headers and propagate them unchanged through the
     service call chain (#184). Do not encode business semantics that belong in
     the path, query, or body.
   - The correlation / flow ID (#233) is the one header every API should support:
     pass it through all services and events for tracing and log correlation.
     Accept UUIDs, base64, or random strings up to 128 chars; generate one if
     none is provided and propagate it to downstream calls.
   - The only sanctioned hop-by-hop exception is the `X-RateLimit-*` family
     (#153).

8. **Signal deprecation with `Deprecation` / `Sunset` (#187, #189).** Reflect a
   deprecated element in the API spec (#187) and, during the deprecation phase,
   add response headers: `Deprecation` (`true` or an `@<unix-timestamp>`) and,
   if scheduled, `Sunset` (an HTTP-date). If several elements are deprecated, set
   both to the earliest applicable time. Do not use `Link rel="sunset|deprecation"`.

## Examples

Conditional update (optimistic locking):

```http
GET /api/v1/orders/42 HTTP/1.1

HTTP/1.1 200 OK
ETag: "5db68c06-1a68-11e9-8341-68f728c1ba70"

PUT /api/v1/orders/42 HTTP/1.1
If-Match: "5db68c06-1a68-11e9-8341-68f728c1ba70"
Content-Type: application/json
```

Deprecation headers on a response:

```http
HTTP/1.1 200 OK
Deprecation: @1758095283
Sunset: Wed, 31 Dec 2025 23:59:59 GMT
```

Reuse standard definitions rather than redefining each header. The plugin
bundles the header schemas at
`../../reference/models/headers-1.0.0.yaml` (mirroring the
guideline's `headers-1.0.0.yaml`) — copy the entries you need into your spec's
`components`, then `$ref` them locally, e.g.:

```yaml
components:
  parameters:
    ETag:          # definition copied from the bundled headers-1.0.0.yaml
      name: ETag
      in: header
      required: false
      schema: { type: string }
  headers:
    ETag:
      $ref: '#/components/parameters/ETag'   # reference within your own document
```

## Result

Produce header usage (or a review of it) in which: header names are correctly
cased; content negotiation uses standard `Accept`/media types; versioning is in
the URL path and never in the media type; `ETag`/`If-Match` guard concurrent
updates where relevant; `Prefer`, `Idempotency-Key`, and any `X-*` headers are
used only per the rules above; and deprecated elements carry `Deprecation`/
`Sunset`. Every supported header is declared in the operation's spec.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`../../reference/http-headers.md`):

- `reference/http-headers.md` — #132/#133 (casing, standard headers), #178
  (`Content-*`), #181 (`Prefer`), #182 (`ETag`/`If-Match`), #183/#184
  (proprietary `X-*` headers), #230 (`Idempotency-Key`), #233 (`X-Flow-ID`)
- `reference/data-formats.md` — #244 (content negotiation)
- `reference/json-guidelines.md` — #172 (standard media types)
- `reference/compatibility.md` — #114/#115 (URL versioning; media-type
  versioning is a violation here)
- `reference/deprecation.md` — #187/#189 (`Deprecation`/`Sunset`)
- `reference/models/headers-1.0.0.yaml` — reusable header definitions
