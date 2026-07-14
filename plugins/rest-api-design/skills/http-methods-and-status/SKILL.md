---
name: http-methods-and-status
description: Concise decision reference for choosing the correct HTTP method and status code when designing REST operations, per the adapted Zalando guidelines. Use when the user asks "which HTTP method should I use?", "which status code should I return?", or is deciding how to model an operation.
type: knowledge
---

# HTTP Methods & Status Codes — Decision Reference

A crisp, self-contained reference for picking the right HTTP method and status
code under the adapted Zalando REST API Guidelines bundled with this plugin. Use
it while designing an operation to decide semantics, safe/idempotent properties,
request/response shape, and which codes to return. This skill does not modify any
API artifact — it informs the decision.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.

## Method selection

Choose the method from what the operation *means* (#148). Fulfill the RFC 9110
safe/idempotent/cacheable properties (#149):

| Method | Semantics | Safe | Idempotent | Cacheable | Request body | Typical success codes |
|---|---|---|---|---|---|---|
| `GET` | Read a single resource or collection | Yes | Yes | Yes | No (use query params) | `200` (`404` if missing) |
| `HEAD` | Like `GET`, headers only (pairs with `ETag`) | Yes | Yes | Yes | No | `200`, `304` |
| `POST` | Create in a collection, or an action-as-resource | No | No¹ | No | Yes | `201`/`200`/`204`, `202`, `207` |
| `PUT` | Replace an **entire** resource (create-or-replace) | No | Yes | No | Yes | `200`/`204` (update), `201` (create), `202` |
| `PATCH` | **Partial** update via a patch document | No | No¹ | No | Yes | `200`/`204`, `202` |
| `DELETE` | Delete a resource | No | Yes | No | Usually no | `200`/`204`, `202` |
| `OPTIONS` | Inspect available methods of an endpoint | Yes | Yes | No | No | `200` (with `Allow`) |

¹ `POST`/`PATCH` are not idempotent by default; design them idempotent with a
conditional key (`If-Match`), a secondary key in the body, or an
`Idempotency-Key` header when retries/duplicates matter (#229/#231/#230).

Rules of thumb:
- Never give `GET` a request body — encode filters in query params; if the query
  is too large, use `POST` documented as "GET with body".
- Prefer `POST` for creation (server owns the id); use `PUT` for creation only
  when the client controls the identifier and passes it in the path.
- `PATCH`: pick exactly one patch media type per endpoint —
  `application/merge-patch+json` (RFC 7396) or `application/json-patch+json`
  (RFC 6902).

## Actions as resources (#138, #141)

URLs are **noun-based and verb-free** — the HTTP method is the only verb. Model
an action as a sub-resource ("put a message in the letter box"):

- Cancel an order → `POST /api/v1/orders/{id}/cancellations` (not `.../cancel`).
- Lock an article → `PUT /api/v1/article-locks/{article-id}` (not `.../lock`).

Benefit: you also get a resource to browse/filter the actions taken.

## Status codes — use only standard, documented codes

Use **only** official HTTP status codes with their RFC-intended semantics
(#243), choose the **most specific** applicable code (#220), and document all
success and service-specific error responses per operation (#151). Do not invent
non-standard codes. Below is the sanctioned subset (#150).

### Success (2xx)

| Code | When | Applies to |
|---|---|---|
| `200 OK` | General success with payload | all |
| `201 Created` | Resource created; return `Location` to it | `POST`, `PUT` |
| `202 Accepted` | Accepted for **async** processing (#253) | `POST`/`PUT`/`PATCH`/`DELETE` |
| `204 No Content` | Success, no response body | `POST`/`PUT`/`PATCH`/`DELETE` |
| `207 Multi-Status` | Per-item results of a batch/bulk request (#152) | `POST` (`DELETE`) |

### Redirection (3xx)

Avoid redirection codes in REST APIs (#251) — `301`/`302`/`303`/`307`/`308` are
**do not use**. The only relevant one is `304 Not Modified`, for conditional
`GET`/`HEAD` with `If-None-Match`/`If-Modified-Since`. For conditional
`PUT`/`PATCH`/`DELETE`, use `412` instead of `304`.

### Client errors (4xx)

| Code | When | Document? |
|---|---|---|
| `400 Bad Request` | Malformed/invalid request; **also** business/semantic validation failures | document |
| `401 Unauthorized` | Missing/invalid credentials (actually *unauthenticated*) | do not document |
| `403 Forbidden` | Authenticated but not permitted (document only object-specific case) | usually not |
| `404 Not Found` | Target resource/entity not found | do not document |
| `405 Method Not Allowed` | Method blocked by resource state (rare; needs a real use case) | document |
| `406 Not Acceptable` | Cannot produce a media type in `Accept` | do not document |
| `409 Conflict` | Conflict with current resource state (e.g. duplicate create) | document |
| `410 Gone` | Resource intentionally/traceably deleted | usually not |
| `412 Precondition Failed` | `If-Match`/`If-None-Match` condition failed (optimistic locking, #182) | do not document |
| `415 Unsupported Media Type` | Unsupported request `Content-Type` | do not document |
| `422 Unprocessable Content` | **Do not use** — use `400` for semantic/validation errors instead | — |
| `423 Locked` | Pessimistic lock (prefer optimistic locking) | document |
| `428 Precondition Required` | Server requires a conditional request (#182) | do not document |
| `429 Too Many Requests` | Rate limit exceeded; send `Retry-After` or `X-RateLimit-*` (#153) | do not document |

### Server errors (5xx)

| Code | When | Document? |
|---|---|---|
| `500 Internal Server Error` | Unexpected server failure | do not document |
| `501 Not Implemented` | Endpoint not yet implemented (planned) | document |
| `503 Service Unavailable` | Temporarily unavailable; send `Retry-After` | do not document |

### Errors are problem+json (#176)

Every endpoint must be able to return errors as **`application/problem+json`**
(RFC 9457) for both `4xx` and `5xx`, carrying `type`/`title`/`status`/`detail`/
`instance`. Never expose stack traces (#177). Use relative-URI problem `type`
identifiers like `/problems/out-of-stock`.

## Quick checklist when designing an operation

1. Pick the method by semantics; confirm safe/idempotent expectations (#148/#149).
2. Keep the URL verb-free — model actions as sub-resources (#141/#138).
3. Enumerate success codes for that method (table above) and document them (#151).
4. Enumerate the client/server errors it can return; wire up `problem+json` (#176).
5. Add locking/creation headers as needed: `Location` on create, `ETag`/`If-Match`
   → `412` for updates (#182).

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/http-requests.md` — #148 method semantics, #149 properties,
  #229/#231 idempotent design, #253 async
- `reference/http-status-codes-and-errors.md` — #150/#151/#243/#220 codes,
  #152 (`207`), #153 (`429`), #176 problem+json, #177, #251 redirects
- `reference/urls.md` — #138/#141 actions-as-resources, #135/#114 base path & versioning
- `reference/http-headers.md` — #182 `ETag`/`If-Match`, #133 `Location`
- `reference/index.md` — chapter map and adaptation summary
