---
name: fix-compliance
description: Apply remediations for findings from a REST API compliance audit, safely partitioning fixes that are spec-internal from fixes that would break the wire contract. Use when the user asks to fix audit findings, remediate compliance issues, or apply the results of an API audit, as opposed to just reviewing them.
type: builder
argument-hint: "[path-or-spec-glob | audit-report-file]"
---

# Fix REST API Compliance Findings

Apply remediations for guideline violations in a REST API, building on the
**`audit`** skill's findings. `audit` deliberately never edits files; this
skill is the follow-up that does — but only for fixes that don't change the
wire contract. Fixes that would break clients are planned, not auto-applied.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

This skill **modifies the API artifact** for the safe subset of findings.
Produce the edited file(s), a migration plan for the rest, and a short summary.

## Arguments

`$ARGUMENTS` (optional) is either:
- A path or glob narrowing which artifact to fix (same convention as `audit`),
  or
- A path to an existing audit report file to read findings from.

If `$ARGUMENTS` points at an existing report, parse its findings directly.
Otherwise, run the **`audit`** skill's Step 1–2 checklist inline first to
produce the findings — say explicitly that you did this before proceeding.

## Step 1 — Detect the API artifact

Same detection as `audit`: look for an OpenAPI document
(`*.yaml`/`*.yml`/`*.json` containing `openapi:`/`swagger:`; common locations
`openapi.yaml`, `api.yaml`, `api/`, `spec/`, `docs/`); degrade to code-first
routes if none exists. State which artifact you are fixing.

## Step 2 — Partition every finding

For **each** finding, decide which bucket it belongs to. When in doubt, put it
in the breaking bucket — an unnecessary migration plan is far cheaper than an
accidental break.

### Safe to apply now (spec-internal, invisible to clients on the wire)

- Missing/incomplete **meta-information**: `info.contact`, `info.x-api-id`
  (#215), `info.x-audience` (#219), missing/incorrect semantic version
  (#116).
- Missing error handling: no `default` / `application/problem+json` response
  defined (#176) — adding a response definition that documents existing
  behavior doesn't change what the server sends.
- Undocumented status codes the implementation already returns (documenting is
  not changing behavior).
- Missing pagination **metadata in the spec** for a response that is *already*
  a JSON object with an `items` array — adding the `self`/`next`/etc. link
  definitions to the schema (#161/#248) when the shape is unchanged.
- Missing `security` declarations for endpoints that the implementation
  already enforces authentication/scopes on (documenting existing behavior).
- Missing header definitions (`ETag`, `Deprecation`, etc.) for headers already
  sent by the implementation.
- Missing or thin `description`/`summary` fields, examples.
- `additionalProperties: false` present where it shouldn't be (#111) —
  **loosening** a constraint (removing the false) is backward compatible for
  clients (it only ever rejected extra fields the server itself allowed
  through), so it is safe to fix. Do not "tighten" the other direction.

### Breaking — do NOT auto-apply; plan instead

- Renaming a camelCase (or other non-conforming) property to snake_case
  (#118) — this changes the wire property name.
- Un-bare-ing a top-level array response into an object (#110) — this changes
  the response shape.
- Adding the `/api`/`v1` prefix to a root-served or otherwise misplaced API
  (#135/#114) — this changes every URL.
- Pluralizing a singular collection name (#134) — changes the URL.
- Converting a verb-based URL into a noun/sub-resource form (#141) — changes
  the URL.
- Any status-code semantic change (e.g. an endpoint currently returns `200`
  where the guidelines call for `201`+`Location`, and clients may depend on
  the current behavior).
- Any change classified breaking by the **`modify-schema`** skill's rules
  (#106/#107): removing/renaming a field, tightening validation, changing a
  field's type, extending a closed output `enum`.

If a finding doesn't clearly match either list, treat it as breaking and
explain why in the summary rather than guessing.

## Step 3 — Apply the safe set

- Apply fixes one category at a time (meta-information, then errors, then
  pagination metadata, then headers, then descriptions), matching the file's
  existing style and reusing existing `components` where present.
- Do not bundle a safe fix and a breaking fix into the same edit even if they
  touch the same operation — keep them separable in the diff.

## Step 4 — Plan the breaking set

For each breaking finding, output a migration step naming which follow-up
skill applies:
- **Field rename/removal/retype/enum change** → route through **`modify-schema`**
  first (it will confirm the compatible near-equivalent: add-new +
  **`deprecate`** the old, or hand off further to `new-version` if a true break
  is unavoidable).
- **URL/path structural change** (base path, versioning, pluralization, verb
  removal) → these are only safe behind a new major version — route through
  **`new-version`**, since the old paths must keep working during migration.
- **Status-code semantic change** → treat like a schema change: introduce the
  corrected behavior behind a new version or a new operation variant, and
  **`deprecate`** the old behavior.

Order the plan so version-introducing changes are batched together (one
`new-version` pass covers many breaking findings at once, not one per finding).

## Step 5 — Re-audit and report deltas

Re-run the `audit` checklist over the edited artifact. Report the finding count
before and after, by severity, so the user can see exactly how much was
resolved automatically versus deferred.

## Verify

1. Re-read every edited file; confirm it still parses (YAML/JSON quick parse,
   e.g. `python -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <file>`).
2. Confirm no `$ref` was left dangling by a safe-set edit.
3. If a spec linter is configured in the project, run it and fix what it flags
   on the touched paths.
4. Confirm the safe-set edits genuinely didn't touch anything from the
   breaking-set list — spot check field names and URLs are unchanged where the
   plan says they should be.

## Output

1. State the detected artifact and where the findings came from (fresh audit
   vs. supplied report).
2. **Findings partition**: a table or list of every finding, tagged safe/
   breaking, with the rule number.
3. The **edited artifact** with the safe-set fixes applied.
4. The **migration plan** for the breaking set: which follow-up skill handles
   each, and a sensible order of operations.
5. The **before/after finding counts** from the re-audit.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/<file>.md`); this mirrors what `audit` cites:

- `reference/urls.md` — #135, #114, #134, #129, #141 (URL/naming findings —
  breaking if changed)
- `reference/compatibility.md` — #106–#108, #110, #111, #113 (schema/response
  shape findings)
- `reference/http-status-codes-and-errors.md`, `reference/http-requests.md` —
  status code and method findings
- `reference/json-guidelines.md`, `reference/data-formats.md` — payload
  findings
- `reference/pagination.md`, `reference/http-headers.md`, `reference/security.md`
- `reference/meta-information.md`, `reference/deprecation.md` — safe-set
  meta-information findings
- `reference/index.md` — chapter map and adaptation summary

Cross-reference: `audit` (produces the findings this skill consumes),
`modify-schema` (applies compatible field-level fixes), `deprecate` (marks
superseded elements during migration), `new-version` (carries breaking
structural fixes forward under a new major version).
