---
name: modify-schema
description: Add, remove, rename, or retype a field on an existing REST API schema, enforcing the compatible-extension rules so breaking changes are caught before they're applied. Use when the user asks to add a field, remove a field, rename a property, change a field's type, or otherwise modify an existing resource's schema.
type: builder
argument-hint: "<schema-or-path> <add|remove|rename|retype> <field>[ <new-name-or-type>]"
---

# Modify a Schema Field

Change a field on an **existing** schema — add, remove, rename, or retype a
property — while enforcing the adapted Zalando compatible-extension rules.
Classify the change first; only apply it if it is compatible, otherwise stop
and hand off to the versioning path.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

This skill **modifies the API artifact** when the change is compatible.
Produce the edited file(s) and a short summary of every change made — or, if
the change is breaking, produce a compatibility verdict and hand off instead
of editing.

## Arguments

`$ARGUMENTS` is free-form; parse it, don't naively split on spaces since a
schema/path or field name may itself matter word-for-word. Expected shape:
`<schema-or-path> <add|remove|rename|retype> <field>[ <new-name-or-type>]`,
e.g. `Order add tracking_number`, `Order rename legacy_status status`,
`/api/v1/orders/{id} retype total number`. `<schema-or-path>` may be a
`components/schemas` name or a resource path — resolve either to the schema in
use. If `$ARGUMENTS` is empty or ambiguous, ask the user which schema/field and
what kind of change.

## Step 1 — Detect the API artifact

1. Look first for an **OpenAPI document**: a `*.yaml`/`*.yml`/`*.json` file
   containing `openapi:` or `swagger:` (common names/locations: `openapi.yaml`,
   `api.yaml`, under `api/`, `spec/`, `docs/`). This is the primary target.
2. If none exists, degrade gracefully to **code-first** models (JPA/Hibernate
   entities, Pydantic/Marshmallow models, TypeScript interfaces/DTOs, etc.) and
   apply the change in that idiom, stating the assumption.
3. **Tell the user which artifact you detected** and are editing.

## Step 2 — Locate the schema and its direction

- Find the target `components/schemas` entry (resolve it from a resource path
  if one was given: find the path's request/response `$ref`).
- Walk every usage of the schema across the spec's `requestBody`s and
  `responses` to determine whether it is used for **input only**, **output
  only**, or **both** — the compatibility rules differ by direction (#107).
  If the same schema serves both directions, the stricter "both" rule set
  applies (the intersection of the input and output rules).

## Step 3 — Classify the change (#106, #107, #111, #112)

**Compatible — safe to apply:**
- Add an **optional** field (input, output, or both).
- Add a **mandatory** field to an **output-only** schema.
- Make a **mandatory input field optional** (never the reverse).
- Make an **optional output field mandatory** (never the reverse).
- Extend an **input-only** `enum`'s accepted range.
- Add values to an extensible enum — one marked `[Extensible enum]` in its
  `description` with an `examples` value list, or using the legacy
  `x-extensible-enum` keyword (any direction, #112).
- Reduce an enum range, but only if the server still accepts and handles the
  old values too.

**Breaking — do NOT apply automatically:**
- Remove any field, from any schema, in any direction (#107 hint: even
  "unused" field removal is treated as non-compatible because it later allows
  a same-named field with a different type/semantics).
- Rename any field (a rename is a remove + unrelated add from the wire's
  perspective).
- Change a field's type or semantics.
- Add a **mandatory** field to an **input** (or both-direction) schema, or make
  an optional input field required.
- Tighten validation (formats, ranges, lengths) on an input field.
- Extend a **closed** `enum`'s range on an **output** (or both-direction)
  schema — clients may not handle the new value.
- Anything on a both-direction schema that isn't allowed by both the input AND
  output rule sets.

## Step 4 — If breaking, STOP and hand off

Do **not** edit the artifact. Report:
1. The **compatibility verdict**: which specific rule (#106/#107/#111) the
   requested change violates, and why.
2. A **compatible near-equivalent** where one exists:
   - *Rename* → add the new field (compatible add), mark the old field
     `deprecated: true` with a migration note, and point to the **`deprecate`**
     skill to wire the `Deprecation`/`Sunset` headers.
   - *Remove* → same pattern: deprecate first, remove only after consumer
     migration and a major version bump.
   - *Retype / tighten validation* → add a **new** field with the new
     type/constraints alongside the old one; deprecate the old one.
3. If no compatible path exists, or the user explicitly wants the breaking
   change now: recommend the **`new-version`** skill (new major URL version,
   #114) or a new resource variant (#113) — do not silently apply the change.

## Step 5 — If compatible, apply the edit

Obey payload conventions while writing the field:
- snake_case matching `^[a-z_][a-z_0-9]*$` (#118); pluralize array properties,
  singular object properties (#120).
- Never a nullable boolean (#122); `null` and absent carry identical semantics
  (#123).
- Date/time fields named `*_at`/`*_date`/`*_time`, RFC 3339 `format: date-time`
  or `date` (#169/#235).
- Enums as `UPPER_SNAKE_CASE` strings (#240); for growth-prone sets prefer an
  extensible enum — `examples` list + `[Extensible enum]` description prefix
  (`x-extensible-enum` only on OpenAPI < 3.1) (#112).
- Money via the shared `{amount, currency}` object, `amount` as
  `format: decimal` (#173) — copy from
  `${CLAUDE_PLUGIN_ROOT}/reference/models/money-1.0.0.yaml#/Money` if not
  already present, `$ref` it locally.
- `readOnly` for response-only fields, `writeOnly` for request-only fields
  (#252).
- Never declare `additionalProperties: false` on the containing object (#111).
- For a new **input** field, do not make it mandatory unless the schema is
  output-only.

## Step 6 — Bump the version number (#116)

- Compatible **additions**: bump `info.version` **MINOR** (`1.2.0` → `1.3.0`).
- Doc-only clarifications (no schema shape change): bump **PATCH**.
- Never touch the URL version segment for a compatible change (#114).

## Verify

1. Re-read the edited file; confirm it still parses (for YAML/JSON, run a
   quick parse, e.g.
   `python -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <file>`).
2. Confirm the `$ref` to the schema (and to `Money` if newly added) resolves
   within the document.
3. If a spec linter is configured in the project (`spectral`, `redocly`,
   `swagger-cli`, `openapi-spec-validator`), run it against the touched schema
   and fix what it flags. Don't install a linter unasked; if none is present,
   say so in the summary.
4. Confirm no other operation's `required` list, `example`, or sibling schema
   now contradicts the edit.

## Output

1. State the detected artifact, the target schema, and its input/output
   direction.
2. The **compatibility verdict** (compatible / breaking) and which rule
   governed the call.
3. If compatible: the **edited artifact** plus a summary of the field change,
   the new `info.version`, and any conventions applied (format, naming, Money).
4. If breaking: **no edit** — the violated rule, the compatible near-equivalent
   offered, and which follow-up skill (`deprecate`, `new-version`) to use.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`):

- `reference/compatibility.md` — #106/#107 (compatible extension rules by
  direction), #111 (open for extension), #112 (extensible enums), #113
  (avoid versioning)
- `reference/json-guidelines.md` — #118 (snake_case), #120 (pluralization),
  #122/#123 (booleans/null), #173 (Money), #240 (enums), #252 (readOnly/writeOnly)
- `reference/data-formats.md` — #169/#235 (date/time formats)
- `reference/meta-information.md` — #116 (semantic versioning of `info.version`)
- `reference/models/money-1.0.0.yaml` — the shared Money schema to `$ref`
- `reference/index.md` — chapter map and adaptation summary
