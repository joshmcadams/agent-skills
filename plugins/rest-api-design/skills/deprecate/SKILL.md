---
name: deprecate
description: Mark an operation, path, field, or whole API version as deprecated in a REST API spec, and wire the Deprecation/Sunset response headers and migration process. Use when the user asks to deprecate an endpoint, field, or version, retire part of an API, or set a sunset date, without necessarily bumping to a new major version.
type: builder
argument-hint: "<operation|path|field|version> [sunset-date]"
---

# Deprecate an API Element

Mark an operation, parameter, schema, property, or an entire API version as
deprecated, and wire the response headers and process that let clients migrate
safely. This is the standalone deprecation workflow — the **`new-version`**
skill also deprecates the old version, but only as a side effect of a major
bump; use this skill whenever you need to deprecate **without** necessarily
introducing a new version (e.g. sunsetting one field, or one endpoint).

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

This skill **modifies the API artifact**. Produce the edited file(s) and a short summary of every change made.

## Arguments

`$ARGUMENTS` is `<target> [sunset-date]`. `<target>` identifies what to
deprecate — treat it as a single token, don't split on internal `/` or `.`:
- An operation: `GET /api/v1/orders/{id}/history`.
- A whole path (all its operations): `/api/v1/orders/{id}/history`.
- A schema property: `Order.legacy_status`.
- An entire major version's path set: `v1`.

`[sunset-date]` (optional) is the planned shutdown date. If omitted, ask the
user for one, or insert a clearly marked placeholder (`TODO: confirm sunset
date`) and say so in the summary — do not silently invent a date.

## Step 1 — Detect the API artifact and resolve the target

1. Look first for an **OpenAPI document** (`*.yaml`/`*.yml`/`*.json` containing
   `openapi:`/`swagger:`; common locations `openapi.yaml`, `api.yaml`, `api/`,
   `spec/`, `docs/`). If none exists, degrade gracefully to code-first routes
   and state the assumption.
2. Resolve `<target>` to the exact operation object(s), parameter, or schema
   property in the artifact. If it does not exist, say so and stop.
3. State which artifact and which concrete element(s) you resolved before
   editing.

## Step 2 — Set the spec markers (#187)

- Set `deprecated: true` on every affected element: the operation object, the
  parameter object, the schema object, or the specific property — whichever
  matches the target's granularity.
- Extend its `description` with:
  1. **Why** it's deprecated (one line).
  2. **What replaces it** (the new field/endpoint/version, or "no replacement
     — feature retired").
  3. **How to migrate** (concrete steps or a pointer to a migration doc).
  4. The **sunset date**, if planned.
- If deprecating an entire version (`v1`), set `deprecated: true` on every
  operation under that version's path set, not just the top-level path item —
  OpenAPI has no single "deprecate this whole version" switch.

## Step 3 — Wire the `Deprecation` and `Sunset` response headers (#189)

- Add both headers to every response of every affected operation:
  - `Deprecation` (RFC 9745) — either `true` (already deprecated, no specific
    timestamp) or a Unix timestamp like `@1758095283` marking when the
    replacement became/becomes available.
  - `Sunset` (RFC 8594) — an HTTP-date, e.g.
    `Wed, 31 Dec 2025 23:59:59 GMT`, present only if a shutdown is actually
    planned.
  - These use **different time formats** — do not reuse one value for both.
  - If **multiple** elements across the artifact are deprecated at once, set
    both headers to the **earliest** timestamp among them, so consumers get
    the shortest, most conservative migration window.
- Define them once as reusable `components/headers` entries and reference them
  from each affected response's `headers` map, rather than duplicating the
  schema per operation. The plugin root (`${CLAUDE_PLUGIN_ROOT}/reference/models/headers-1.0.0.yaml`)
  is not reachable from the user's own repo, so **copy** the `Deprecation` and
  `Sunset` header definitions into the spec's own `components/headers`, then
  `$ref` them locally:

  ```yaml
  components:
    headers:
      Deprecation:   # copied from the bundled headers-1.0.0.yaml — keep in sync
        required: false
        description: Timestamp or `true` indicating this feature is deprecated.
        schema:
          type: string
          format: date-timestamp
        example: "@1758093035"
      Sunset:
        required: false
        description: HTTP-date at which this feature becomes unavailable.
        schema:
          type: string
          format: http-date
        example: "Wed, 31 Dec 2025 23:59:59 GMT"

  paths:
    /api/v1/orders/{id}/history:
      get:
        deprecated: true
        responses:
          '200':
            headers:
              Deprecation:
                $ref: '#/components/headers/Deprecation'
              Sunset:
                $ref: '#/components/headers/Sunset'
  ```
- Do **not** use `Link: rel="sunset"|"deprecation"` headers or the `Warning`
  header — both are explicitly discouraged in favor of the spec markers plus
  these two headers.

## Step 4 — Process reminder (do not skip)

Headers and spec markers alone are **not sufficient** to shut anything down.
Output this checklist alongside the edit:
- [ ] Obtain **client consent** on the sunset date before any shutdown (#185).
      For external partners, respect the agreed after-deprecation life span
      before they may even start using the feature (#186).
- [ ] **Monitor usage** of the deprecated element in production to track
      migration progress and avoid an uncontrolled break (#188).
- [ ] Clients **must not newly adopt** a deprecated element (#191) — flag this
      in API docs / onboarding material if applicable.
- [ ] Shut down only **after** consumers have migrated, not on the sunset date
      alone if usage monitoring shows stragglers.

## Verify

1. Re-read the edited file; confirm it still parses (YAML/JSON quick parse,
   e.g. `python -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <file>`).
2. Confirm every new `$ref` to `components/headers/Deprecation`/`Sunset`
   resolves within the document.
3. Confirm `deprecated: true` was applied at the correct granularity (not
   accidentally deprecating a sibling operation or the whole path when only
   one method was targeted).
4. If a spec linter is configured in the project, run it and fix flagged
   issues on the touched paths.

## Output

1. State the detected artifact and the resolved target(s).
2. The **edited artifact**: `deprecated: true` markers with description/
   migration text, and the `Deprecation`/`Sunset` headers wired on every
   affected response.
3. The **process checklist** from Step 4, restated as action items for the
   user (consent, monitoring, no-new-onboarding, delayed shutdown).
4. If no sunset date was given, flag the placeholder you inserted (or that you
   asked and are waiting on an answer).

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/deprecation.md` — #187 (spec markers), #189 (`Deprecation`/
  `Sunset` headers), #185/#186 (client/partner consent), #188 (usage
  monitoring), #191 (no new adoption)
- `reference/models/headers-1.0.0.yaml` — `Deprecation`/`Sunset` header schemas
  to copy in and `$ref` locally
- `reference/compatibility.md` — #113/#114 (when deprecation precedes a new
  major version — see the `new-version` skill for that case)
- `reference/index.md` — chapter map and adaptation summary
