---
name: error-responses
description: Design REST API error responses using RFC 9457 application/problem+json. Use when designing error responses, choosing an error format or problem-details schema, deciding which HTTP status code an error should return, or standardizing how an API reports failures.
type: knowledge
---

# Design Error Responses (Problem JSON)

Design consistent, machine- and human-readable error responses for a REST API
using the RFC 9457 **Problem JSON** object (media type
`application/problem+json`). This is a knowledge skill: it produces error-schema
guidance and a designed problem schema, not edits to an artifact.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

## Rules

### Use Problem JSON as the common error format
- **Every** endpoint MUST be able to return a Problem JSON object for client
  errors (`4xx`) and server errors (`5xx`), using media type
  **`application/problem+json`** (#176), per RFC 9457.
- Use **one consistent** error schema across the whole API — do not invent a
  different error shape per endpoint.
- The **source of truth** for the schema is the bundled model at
  `${CLAUDE_PLUGIN_ROOT}/reference/models/problem-1.0.1.yaml#/Problem` (the
  `reference/models/` directory at the plugin root). Since the plugin root is
  not reachable from the user's own repo, **copy** the `Problem` schema into
  the spec's own `components/schemas`, then `$ref` it **locally** so the
  artifact stays self-contained. In an OpenAPI spec, wire it in via the
  `default` response so standard errors need no per-operation duplication:
  ```yaml
  components:
    schemas:
      Problem:   # copied from the bundled problem-1.0.1.yaml — keep in sync with that file
        type: object
        properties:
          type: { type: string, format: uri-reference }
          title: { type: string }
          status: { type: integer }
          detail: { type: string }
          instance: { type: string, format: uri-reference }
  responses:
    default:
      description: error occurred - see status code and problem object.
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'   # reference within your own document
  ```
- **Hint:** `application/problem+json` is often NOT treated as a subset of
  `application/json` by libraries. Clients that want the extended failure info
  should include `application/problem+json` in their `Accept` header, and must be
  robust if a bare status code (no problem body) comes back from infrastructure
  (#176).

### The Problem object fields (#176)
- `type` — a URI **reference** that identifies the problem type. Use a **relative**
  reference like `/problems/out-of-stock`; it is a stable identifier and is **NOT**
  meant to be dereferenced or resolved to a doc page. Defaults to `about:blank`.
- `title` — short, stable, English summary of the problem type, for engineers.
- `status` — the HTTP status code (integer), matching the response status.
- `detail` — human-readable explanation specific to **this occurrence**, in
  English, for engineers.
- `instance` — a URI reference identifying this specific occurrence (e.g.
  `/problems/out-of-stock#order-4711`).

**Extend the object** with your own top-level fields when clients need more
detail (e.g. `out_of_stock_items`, `retry_after_seconds`, a per-field
`validation_errors` array). Custom problem types are extensions of the base
Problem object; keep extension property names `snake_case` (#118). Do not
declare `additionalProperties: false`.

### Map problems to the correct status code (#243, #220, #150)
Use the **most specific** official status code (#220), consistent with RFC
semantics. Common mappings:
- **400 Bad Request** — malformed request, invalid input, **and** input that
  fails business/semantic validation. This is the default for validation errors.
- **401 Unauthorized** — missing/invalid credentials (actually *unauthenticated*).
- **403 Forbidden** — authenticated but not permitted (missing scope / object-level deny).
- **404 Not Found** — target resource does not exist.
- **409 Conflict** — request conflicts with current resource state (e.g. version
  conflict on update). For failed `If-*` preconditions use **412** instead, not 409.
- **410 Gone** — resource intentionally removed and will not return.
- **422 Unprocessable Content** — **discouraged by this guideline**: prefer
  **400** for validation/business-logic failures (#150, #220). Do not use 422 as
  a default; it is listed here only for completeness.
- **429 Too Many Requests** — rate limit exceeded; include `Retry-After` or the
  `X-RateLimit-*` headers (#153).
- **500 Internal Server Error** — unexpected server fault.
- **503 Service Unavailable** — temporary outage; set `Retry-After` when known.

### Do NOT leak internals (#177)
- Never put stack traces, exception class names, SQL, file paths, or internal
  hostnames in `detail` or any field. They leak implementation details and
  vulnerabilities and are not part of the contract.
- `detail`/`title` describe the failure functionally, not the code path.

### Localization (careful)
- The Problem object's `title` and `detail` are **English, engineer-facing, and
  NOT localized**. Do not translate them.
- If you need an end-user-facing localized message, carry it in a **separate
  extension field** (e.g. `user_message` plus a `locale`), negotiated explicitly
  — never repurpose `title`/`detail` for localized end-user text.

## Concrete example
`409 Conflict` returned from `PATCH /api/v1/orders/4711` when stock is
insufficient, with a domain-specific extension field:

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json
```
```json
{
  "type": "/problems/out-of-stock",
  "title": "Item is out of stock",
  "status": 409,
  "detail": "Order 4711 cannot be fulfilled: 2 of the requested items are no longer in stock.",
  "instance": "/problems/out-of-stock#order-4711",
  "out_of_stock_items": ["sku-12", "sku-98"]
}
```

## Output / result
Produce: (1) the recommended **Problem JSON schema** (base object plus any
domain-specific extension fields), (2) the **status-code-to-problem-type
mapping** for the endpoints in question, and (3) notes on the `Accept`-header
behavior and localization approach. Advise, do not silently rewrite the user's
spec unless they ask for the edit.

## Reference
For full detail, see the guidelines bundled with this plugin (the `reference/`
directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/http-status-codes-and-errors.md` — #176 (problem JSON), #177 (no
  stack traces), #150/#220/#243 (status code selection), #153 (429/rate limits)
- `reference/models/problem-1.0.1.yaml` — the Problem object schema to `$ref`
- `reference/json-guidelines.md` — #118 (snake_case property names), #172 (media types)
