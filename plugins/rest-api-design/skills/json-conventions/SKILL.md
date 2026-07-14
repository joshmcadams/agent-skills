---
name: json-conventions
description: Conventions for designing JSON request/response payloads, schemas, property names, and data types in a REST API following the adapted Zalando guidelines. Use when designing JSON request/response bodies, defining schemas or models, choosing property names, or picking standard data types/formats (dates, money, enums, codes).
type: knowledge
---

# JSON Payload & Data-Type Conventions

Apply these rules when designing JSON request/response payloads and their
schemas. Each rule is distilled inline so this skill is usable on its own; cite
the bundled chapters only for full detail.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.

This is a **knowledge** skill — it produces guidance and reference material, not edits to an artifact. Advise; do not modify files unless asked.
## Payload structure

- [ ] **Top-level response bodies MUST be a JSON object**, never a bare array or
  a map (#110). This leaves room to add fields (e.g. pagination metadata) later
  without breaking clients. Wrap a list as `{ "items": [...] }`, not `[...]`.
- [ ] Use JSON as the interchange format; payloads are UTF-8, valid Unicode,
  with unique member names (#167).
- [ ] Design a **single schema for reading and writing** the same resource;
  mark request-only fields `writeOnly` (e.g. passwords) and response-only
  fields `readOnly` (e.g. ids) (#252).

## Property naming

- [ ] Property names MUST be **snake_case**, never camelCase — regex
  `^[a-z_][a-z_0-9]*$` (#118). E.g. `customer_number`, `billing_address`.
- [ ] Be **consistent**: same name + same semantics across the whole API; use
  common field names — `id` (opaque **string**, never a number), `xyz_id` for
  references (`parent_node_id`), `etag` (#116/#117/#174).
- [ ] **Pluralize array property names**, keep object names singular (#120):
  `items`, `addresses` (array) vs `address` (object).
- [ ] Date/time property names contain `date`/`time`/`timestamp` or end with
  `_at`: `created_at`, `modified_at`, `arrival_date` (#235).

## null, absent, empty

- [ ] Treat **`null` and absent with identical semantics** — do not assign
  different meanings to `{}` vs `{"x":null}` (#123). To model a required-but-
  unset value, prefer a Null object / dedicated status over a bare `null`.
- [ ] **Never use `null` for a boolean** (#122). If a boolean has a meaningful
  third state, replace it with an enum, e.g.
  `accepted_terms: YES | NO | UNDEFINED`.
- [ ] **Empty arrays are `[]`, never `null`** (#124).

## Enums

- [ ] Declare enums as `string`-typed with **`UPPER_SNAKE_CASE`** values, e.g.
  `VALUE`, `YET_ANOTHER_VALUE` (#240).
- [ ] Prefer **extensible enums** (`x-extensible-enum`) for value sets that may
  grow, so adding a value stays backward-compatible (#112).
- [ ] Exception: case-sensitive values sourced outside the spec (e.g. ISO
  language codes) keep their native casing.

## Standard data formats

Always pick a standard OpenAPI `format` so clients never guess precision (#238).

- [ ] **Numbers/integers MUST declare a format** (#171): integers use `int32` /
  `int64` / `bigint`; numbers use `float` / `double` / `decimal`.
- [ ] **Dates & times** use RFC 3339 string formats (#169/#238): `date`
  (`"2019-07-30"`), `time`, `date-time` (`"2019-07-30T06:43:40.252Z"`), plus
  `time-local` / `date-time-local` (complement with a `tz-id` field). Use
  upper-case `T` and `Z`; prefer UTC without offset; avoid numeric timestamps
  (#126/#127). Pick `date-time` for an exact instant, `date` for a day.
- [ ] **Duration / interval**: ISO 8601 `duration` (`"P1DT3H4S"`) and `period`
  (`"2022-06-30T14:52:44/PT48H"`) string formats (#127). Name an interval query
  param `<time>_between` and an interval property `<time>_interval`.
- [ ] **Country / language / currency** codes (#170/#128): `iso-3166-alpha-2`
  (`"GB"`), `iso-639-1` (`"en"`) or `bcp47` (`"en-DE"`), `iso-4217` (`"EUR"`).
- [ ] **Binary data** is a `string` with `binary` format, `base64url`-encoded
  (#239); use a proper standard media type where one exists.
- [ ] **IDs are opaque strings**; use `uuid` format only when large-scale,
  uncoordinated id generation is actually needed (#144).

## Money

Use the common **Money** object (#173): an `amount` plus a `currency` — never a
naked number. `amount` is a JSON `number` with the custom **`format: decimal`**
(a non-standard format adopted precisely so clients handle it as an exact
decimal, e.g. Java `BigDecimal` — **never** deserialize to `float`/`double`, or
you lose precision). `currency` uses `iso-4217`. Both are required.

```yaml
Money:
  type: object
  properties:
    amount:   { type: number, format: decimal, example: 99.95 }   # bare number, not "99.95"
    currency: { type: string, format: iso-4217, example: EUR }
  required: [amount, currency]
```

Money is a **closed** type — do not extend it via inheritance (no
`discounted_amount` alongside `amount`). Favor composition:

```json
{
  "price":            { "amount": 19.99, "currency": "EUR" },
  "discounted_price": { "amount": 9.99,  "currency": "EUR" }
}
```

Reference the bundled model instead of re-declaring it:

```yaml
grand_total:
  $ref: '${CLAUDE_PLUGIN_ROOT}/reference/models/money-1.0.0.yaml#/Money'
```

## Worked example

```json
{
  "id": "ord_9f2c",
  "status": "SHIPPED",
  "created_at": "2026-07-14T09:30:00Z",
  "shipping_country_code": "DE",
  "is_gift": false,
  "grand_total": { "amount": 42.20, "currency": "EUR" },
  "line_items": []
}
```

Note: object top level (#110), snake_case names (#118), string opaque `id`,
`UPPER_SNAKE_CASE` enum `status` (#240), RFC 3339 `created_at` (#169),
`iso-3166-alpha-2` country code (#170), boolean (not null) `is_gift` (#122),
Money object (#173), pluralized empty array `line_items: []` (#120/#124).

## Reference

For full detail, see the guidelines bundled with this plugin (the `reference/`
directory at the plugin root, e.g. `${CLAUDE_PLUGIN_ROOT}/reference/<file>`):

- `reference/json-guidelines.md` — #110, #118, #120, #122–#124, #167, #172,
  #173 (Money), #174, #235, #240, #252
- `reference/data-formats.md` — #238 (format table), #171, #169/#126, #127,
  #170/#128, #144, #239
- `reference/models/money-1.0.0.yaml` — the shared Money schema to `$ref`
