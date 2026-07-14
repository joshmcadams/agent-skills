---
name: api-compatibility
description: Evolve a REST API without breaking clients. Use when changing, extending, or versioning an API, deciding whether a change is breaking, planning a new major version, or deprecating an endpoint, field, or version with Deprecation/Sunset headers.
type: knowledge
---

# Evolve an API Without Breaking Clients

Guidance for changing a REST API safely: prefer backward-compatible extension,
recognize breaking changes, and — only when a break is unavoidable — introduce a
new **URL** major version and deprecate the old one. This is a knowledge skill:
it produces a compatibility decision and a migration/deprecation plan, not edits
to an artifact.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

## Core principle: don't break backward compatibility (#106)
APIs are contracts; consumers have independent release cycles. Keep all
consumers running. Two techniques: (1) compatible extension, and (2) new
versions with a deprecation process. **Strongly prefer (1)** — avoid versioning
wherever possible (#113).

## Rules for compatible extension

### Provider side (#107)
- Never change the **semantics** of an existing field.
- **Output** schemas: you MAY add fields (optional or mandatory); you MAY make
  optional fields mandatory; you MUST NOT remove fields; `enum` output ranges MAY
  be reduced but MUST NOT be extended (clients may not handle new values).
- **Input** schemas: you MAY add **optional** fields (never mandatory); you MAY
  make mandatory fields optional (not vice-versa); you MUST NOT remove fields;
  never make validation **more** restrictive; `enum` input ranges MAY be extended
  and MAY be reduced only if the server still accepts old values.
- Schemas used for **both** input and output: apply the intersection — add only
  optional fields, remove nothing, flip no required-ness, don't tighten
  validation, don't extend enums.
- Treat OpenAPI objects as **open for extension** — never declare
  `additionalProperties: false` (#111).
- Prefer open-ended/extensible enums (via `examples`, or legacy
  `x-extensible-enum`) over closed `enum` for values likely to grow (#112).
- Always return a **JSON object** (never an array/map) as the top-level response
  structure, so you can add fields (e.g. pagination) later (#110).

### Consumer side — tolerant reader / Postel's Law (#108)
- Be **tolerant** reading responses: ignore unknown fields, but don't strip them
  from a payload you echo back in a subsequent `PUT`.
- Be prepared for **extensible enum** return values you don't recognize — provide
  default/fallback behavior; don't map straight onto a closed language `enum`.
- Be prepared for **HTTP status codes** not explicitly documented (handle them
  like the corresponding `x00`). Follow a `301` redirect.
- Be **conservative** in requests: don't exploit unspecified limits.

### Provider conservatism (#109)
- Servers SHOULD **reject** unknown input fields with `400` (a deliberate
  deviation from Postel's Law) rather than silently ignore them — silent
  ignoring hides client typos and blocks future compatible extension. If you do
  accept unknown fields, document how they are handled for `PUT`/`POST`/`PATCH`.
- Define input constraints (formats, ranges, lengths) accurately and validate them.

## What counts as a breaking (incompatible) change
Any of these breaks consumers and requires a new major version + deprecation:
- Removing or renaming a field, resource, or endpoint.
- Changing a field's type or semantics.
- Adding a **mandatory** input field, or making an optional input field required.
- Tightening validation / reducing accepted input ranges.
- Extending an `enum` used in **output** (closed enum), or removing an output field.
- Changing the meaning of an existing status code or removing a supported one.

## CRITICAL DEVIATION: versioning a genuine break
When — and only when — a change cannot be made compatibly (#113):
- Introduce a new **major version in the URL path**: `/api/v2/...` (#114). The
  segment is `v` + a positive integer, no leading zeros, **major only**. Minor,
  backward-compatible changes MUST NOT change the URL.
- Keep the media type as plain **`application/json`** — do **NOT** encode the
  version in `Accept`/`Content-Type` (#115). Media-type versioning is a violation.
- **Run old and new major versions in parallel** by the same service; migrate
  consumers off the old version via the deprecation process. Before minting a new
  URL version, first consider the compatible alternatives (a new resource variant,
  or a new service) (#113).

## Deprecation process (#187, #189, #185)
- Reflect deprecation **in the API spec**: set `deprecated: true` on the affected
  operation / parameter / schema / property and explain in `description`, with a
  planned sunset date and migration advice (#187).
- On each affected response, add response headers (#189):
  - `Deprecation: <timestamp>` (RFC 9745) — a timestamp like `@1758095283`, or
    `true` if already deprecated.
  - `Sunset: <http-date>` (RFC 8594) — e.g. `Wed, 31 Dec 2025 23:59:59 GMT`.
  - If multiple elements are deprecated, use the **earliest** timestamp.
  - Reference the bundled header definitions:
    `${CLAUDE_PLUGIN_ROOT}/reference/models/headers-1.0.0.yaml#/Deprecation` and
    `.../#/Sunset`.
- Headers alone are **not** sufficient: obtain client consent on the sunset date,
  monitor usage of the deprecated version, and only shut down once consumers have
  migrated (#185, #188). Clients MUST NOT start using a deprecated API (#191).

## Decision checklist
1. Can the change be made compatibly (see provider rules #107/#111/#110)?
   → **Yes:** extend in place. No URL/version change. Ensure clients are tolerant
     readers (#108). Done.
2. Is it breaking (see the breaking-change list)?
   → **Yes:** prefer a compatible alternative or new resource variant (#113) if
     feasible; otherwise mint a **new URL major version** `/api/v2/...` (#114,
     NOT media-type versioning #115), run it **in parallel** with the old, and
     **deprecate** the old version — `deprecated: true` in the spec plus
     `Deprecation`/`Sunset` headers (#187/#189) — with client consent and usage
     monitoring before sunset.

## Output / result
Produce: a **compatibility verdict** (compatible extension vs. breaking), the
concrete change plan (fields/endpoints affected), and — if breaking — the new
URL version + a **deprecation/migration plan** (spec markers, headers, timeline,
consent, monitoring). Advise; don't rewrite the spec unless asked.

## Reference
For full detail, see the guidelines bundled with this plugin (the `reference/`
directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/compatibility.md` — #106–#113 (compatible extension, tolerant
  reader), #114 (URL versioning), #115 (no media-type versioning)
- `reference/deprecation.md` — #187/#189 (spec markers, `Deprecation`/`Sunset`
  headers), #185/#188/#191 (consent, monitoring, shutdown)
- `reference/models/headers-1.0.0.yaml` — `Deprecation` / `Sunset` header schemas
