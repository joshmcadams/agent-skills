---
name: api-first
description: The API-First principle and how APIs must be specified — define the API with an OpenAPI specification before implementing, keep it a single self-contained document, prefer OpenAPI 3.1, use U.S. English, and review external-facing APIs. Use when the user asks how to start or define an API, whether to go spec-first or code-first, which OpenAPI version to use, how to structure/split spec files, or about the API review/publishing process.
type: knowledge
---

# API-First & API Specification

The foundational rules for *how* an API is defined, before any of the design
details (URLs, schemas, status codes) matter: define the API first with an
OpenAPI specification, keep that spec self-contained, and review it. These are
`MUST` rules — an API that skips them is non-compliant no matter how clean its
URLs are, so this is the first thing to get right on a new API and the first
thing the `audit` skill checks.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

This is a **knowledge** skill — it produces guidance and reference material, not edits to an artifact. Advise; do not modify files unless asked. To scaffold a compliant spec, use **create-new-api**; to check an existing repo against these rules, use **audit**.

## The API-First principle (#100)

Follow API First — three concrete obligations:

- **Define the API before implementing it**, using OpenAPI as the
  specification language (#101). The spec is the contract and the source of
  truth; the implementation follows the spec, not the other way around. A
  code-first service whose "spec" is auto-generated from handlers *after* the
  fact inverts this and is a violation.
- **Design consistently with these guidelines**, and use an **API linter**
  (e.g. Spectral, Zally, or an equivalent) for automated rule checks rather
  than relying on manual review alone.
- **Get early review feedback** from peers and client developers. Apply a
  lightweight API review process for every **component-external** API — any API
  whose `x-audience` is not `component-internal` (see the `create-new-api` and
  `api-security` skills for the audience values, #219). Internal-only APIs can
  be reviewed more lightly.

Spec-first is worth the discipline because it lets clients and servers be
designed in parallel against an agreed contract, surfaces design problems
before code is written, and makes the guidelines mechanically checkable.

## Provide the specification using OpenAPI (#101)

- Use **OpenAPI** (from the OpenAPI Initiative) as the definition language.
- Provide it as a **single, self-contained file** for readability — one YAML
  document per API rather than many fragments the reader must stitch together.
- Prefer **OpenAPI 3.1** for new APIs. 3.0 and 2.0 (Swagger) still work, but
  3.1 is encouraged — its main change is that Schema Objects are now fully
  standard JSON Schema, which removes some OpenAPI-specific keyword overrides
  (`nullable`, singular `example`, etc.). This is why several skills here
  prefer the plural `examples` list over legacy keywords (see
  `json-conventions` and rule #112).
- Keep the spec under **source control** and **publish** it so consumers can
  find it (#192). Treat spec changes with the same version discipline as code.
- Scope note: these guidelines target resource-oriented HTTP/REST. GraphQL is
  not covered — it can complement REST for backend-for-frontend / mobile
  aggregation cases, but is out of scope for general service-to-service APIs
  here.

## Keep the spec self-contained (#234)

- Specs should be **self-contained by default**: avoid `$ref`s to arbitrary
  local or remote content (`../fragment.yaml#/Element`, or a `$ref` to a
  mutable GitHub blob URL). Such targets are generally **neither durable nor
  immutable** — the referenced content can change or move, silently changing
  your API's meaning or breaking the reference.
- When you need a shared fragment (the `Problem` error object, the `Money`
  object, `Deprecation`/`Sunset` headers), **copy it into your own
  `components/` and `$ref` it locally** — this is the copy-in convention the
  `error-responses`, `json-conventions`, `deprecate`, and `performance` skills
  all use, precisely so the artifact stays self-contained and durable.
- Only reference remote fragments you control and can guarantee are durable
  and immutable.

## Write APIs in U.S. English (#103)

All names, descriptions, and documentation in the spec use **U.S. English**
spelling and vocabulary (e.g. `color`, `canceled`) — consistently across the
whole API.

## Provide an API user manual (#102, SHOULD)

Beyond the spec, it is good practice to provide a **user manual** to improve
client developer experience — covering scope/purpose/use cases, concrete usage
examples, edge cases and error-repair hints, and architecture context. Publish
it online and link it from the spec via `#/externalDocs/url`.

## Output / result

Produce: a clear statement of whether the situation is API-First compliant
(spec exists, spec-first, self-contained, right OpenAPI version, reviewed),
and concrete next actions where it is not — e.g. "generate an OpenAPI 3.1 spec
with `create-new-api`", "inline these external `$ref`s", "set up an API
linter", "add the review step for this external-audience API". Advise; do not
modify files unless asked.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/general-guidelines.md` — #100 (API First principle), #101 (provide
  OpenAPI), #102 (user manual), #103 (U.S. English), #234 (durable/immutable,
  self-contained references)
- `reference/meta-information.md` — #116 (spec vs OpenAPI version), #219
  (`x-audience`, which drives the review obligation)
- `reference/compatibility.md` — #112 (extensible enums via `examples`, an
  OpenAPI 3.1 reason to prefer 3.1)
- `reference/index.md` — chapter map and adaptation summary

Related skills: **create-new-api** (scaffolds a compliant OpenAPI 3.1 spec),
**audit** (checks a repo against #100/#101/#234 among others),
**json-conventions** and **error-responses** (the self-contained copy-in
pattern for shared fragments).
