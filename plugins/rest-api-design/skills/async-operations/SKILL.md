---
name: async-operations
description: Design long-running/asynchronous operations, batch or bulk requests, idempotent retries, and rate limiting for a REST API. Use when the user asks how to model a long-running operation, an async job, a 202 response, a job/status resource, batch or bulk endpoints, idempotency keys, or rate limiting with 429.
type: knowledge
---

# Asynchronous Operations, Batch/Bulk, Idempotency, and Rate Limiting

Apply these rules when an operation cannot (or should not) complete
synchronously, when a single request needs to act on many items at once, when
retries must not create duplicates, or when a service needs to protect itself
from excessive request rates. This is a knowledge skill: it produces design
guidance, not edits to an artifact. To wire the result into a spec, use
**add-endpoint** or **modify-schema** afterward.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

This is a **knowledge** skill — it produces guidance and reference material, not edits to an artifact. Advise; do not modify files unless asked.

## Asynchronous request processing (#253, #181)

REST is synchronous by default; use asynchronous design only for genuinely
long-running work.

- **Preferred pattern — explicit job resource.** Model the async work as its
  own resource, distinct from the business resource it produces:
  1. `POST /api/v1/report-jobs` → `201 Created`, `Location:
     /api/v1/report-jobs/{job-id}`, body carries the `job-id` and an initial
     status.
  2. `GET /api/v1/report-jobs/{job-id}` → `200`, polls the job's status
     (`PENDING`/`RUNNING`/`SUCCEEDED`/`FAILED`, `UPPER_SNAKE_CASE` per #240).
  3. Once `SUCCEEDED`, the job resource carries (or points to) the result id;
     fetch the actual result via its own resource, e.g.
     `GET /api/v1/reports/{report-id}` → `200`.
- **Simpler alternative — no separate job resource.** `POST /api/v1/reports` →
  `202 Accepted` with `Location` pointing at the eventual result resource.
  `GET` on that `Location` returns `202` (still processing) until the result is
  ready, then `200` with the resource. Use this only when you don't need
  richer job status than "not ready yet" / "ready."
- **Client opt-in signal:** support `Prefer: respond-async` (RFC 7240, #181)
  to let a client ask for `202` instead of waiting synchronously; echo
  `Preference-Applied` when honored. Don't force async on every client — make
  it a preference, not the only mode, unless the operation is inherently
  long-running.
- Async applies to `POST`, `PUT`, `PATCH`, and `DELETE` alike — any of them may
  return `202` when processing won't finish within the request.

## Idempotent retries for POST/PATCH (#229, #230, #231)

`POST` and `PATCH` are **not idempotent by default**. Design them idempotent
whenever duplicate submissions or lost updates are a real risk (parallel
clients, retry-on-timeout). Pick from three patterns — they compose:

| Pattern | Applies to | Prevents duplicates | Prevents lost updates | Needs prior GET |
|---|---|---|---|---|
| **Conditional key** (`If-Match`, #182) | `PATCH`/`PUT`/`DELETE` | Yes | Yes | Yes |
| **Secondary key** (body field, #231) | `POST` | Yes | No | No |
| **Idempotency key** (`Idempotency-Key` header) | `POST`/`PATCH` | No (returns identical response) | No | No |

- **Conditional key** — client sends `If-Match: <etag>` with the resource
  version it last read; a stale version fails with `412`. Standardized, HTTP-
  native, inspectable by intermediaries.
- **Secondary key** (preferred for idempotent creation, #231) — a business key
  the client already has (e.g. the shopping-cart id an order is created from),
  stored permanently on the created resource under a uniqueness constraint.
  Retrying the same `POST` should fail `409 Conflict` (or, if you choose to
  treat it as a safe no-op, `200`/`204` returning the *original* resource) —
  don't silently return `201` twice.
- **Idempotency key** — client generates a random `Idempotency-Key` (UUIDv4
  recommended) and resends it on retry; the server returns the exact original
  response without re-executing. Not part of the resource itself, expires
  (24h is the plugin's convention, matching the bundled header model) —
  clients must not rely on it beyond that window. Use this when byte-identical
  responses matter more than HTTP-standard conflict detection.
- Prefer conditional key + secondary key for safe retries before reaching for
  the idempotency key pattern; combine patterns if you need more than one
  guarantee.
- Successful idempotent `POST`/`PATCH` on retry returns `200`/`204` (already
  applied) vs. `201` (first-time creation) — this lets clients tell repetition
  apart from actual creation.

## Batch / bulk requests (#152)

When performance requires acting on many items in a single `POST` (rarely
`DELETE`):

- The endpoint **always** responds `207 Multi-Status` unless the failure is
  **non-item-specific** (overload, auth failure for the whole request) — in
  that case use a normal `4xx`/`5xx` instead.
- A `207` response body carries **per-item** status/result objects, e.g.:
  ```json
  {
    "items": [
      { "id": "itm-1", "status": 201, "location": "/api/v1/orders/itm-1" },
      { "id": "itm-2", "status": 400, "problem": { "type": "/problems/invalid-sku", "title": "Invalid SKU", "status": 400 } }
    ]
  }
  ```
- Wrap the batch response as a top-level JSON **object** with an `items` array
  (#110/#120) — never a bare array, same rule as any other response.
- Document the batch endpoint's atomicity explicitly (all-or-nothing vs.
  best-effort per item) — HTTP gives no default semantics here.

## Rate limiting (#153)

- Return **`429 Too Many Requests`** when a client exceeds its allotted rate.
- Attach one of:
  - **`Retry-After`** — seconds to wait (preferred) or an HTTP-date.
  - The **`X-RateLimit-*`** trio — `X-RateLimit-Limit` (requests allowed per
    window), `X-RateLimit-Remaining` (requests left in the current window),
    and a reset indicator for when the window rolls over.
- `429` is a standard error response — pair it with the usual
  `application/problem+json` body (see the `error-responses` skill) rather
  than inventing a bespoke rate-limit error shape.
- Document `429` on every rate-limited operation; clients must be prepared to
  back off and retry per `Retry-After`.

## Worked example — async report with job resource

```yaml
paths:
  /api/v1/report-jobs:
    post:
      summary: Start an asynchronous report generation job
      responses:
        '201':
          description: Job accepted.
          headers:
            Location: { schema: { type: string, format: uri } }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ReportJob' }
  /api/v1/report-jobs/{job-id}:
    get:
      summary: Poll report job status
      responses:
        '200':
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ReportJob' }
components:
  schemas:
    ReportJob:
      type: object
      properties:
        id:         { type: string, readOnly: true }
        status:     { type: string, examples: [PENDING, RUNNING, SUCCEEDED, FAILED] }
        report_id:  { type: string, description: Set once status is SUCCEEDED. }
      required: [id, status]
```

## Output / result

Produce: (1) the chosen async pattern (job resource vs. bare `202`) with the
concrete paths and status codes, (2) the idempotency pattern(s) selected for
each mutating operation and why, (3) the batch/bulk response shape if
applicable, and (4) the rate-limit header choice. Advise; edit the artifact
via `add-endpoint` or `modify-schema` if the user asks you to apply it.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/http-requests.md` — #229 (idempotent POST/PATCH design), #230
  (`Idempotency-Key`), #231 (secondary key), #253 (async processing)
- `reference/http-status-codes-and-errors.md` — #152 (207 batch/bulk), #153
  (429 rate limits)
- `reference/http-headers.md` — #181 (`Prefer`/`Preference-Applied`), #182
  (`ETag`/`If-Match` conditional key)
- `reference/json-guidelines.md` — #110/#120 (object top-level, pluralization)
- `reference/models/headers-1.0.0.yaml` — `Idempotency-Key`, `Prefer` header
  definitions to copy in
- `reference/index.md` — chapter map and adaptation summary
