# Implementation Plan: New & Extended Skills for the rest-api-design Plugin

This plan is self-contained. It tells an implementing agent exactly what to
build, where, and how to verify it — without needing the analysis conversation
that produced it.

## Background & repo layout

This plugin (`agent-skills/plugins/rest-api-design/`) packages an **adapted
Zalando REST API guideline set** as Claude Code skills:

```
plugins/rest-api-design/
├── .claude-plugin/plugin.json     # plugin metadata (version 1.0.0)
├── ADAPTATION.md                  # the two deviations from upstream Zalando
├── reference/                     # guideline chapters as markdown, rules anchored {#rule-NNN}
│   ├── index.md, urls.md, compatibility.md, deprecation.md, security.md,
│   ├── http-requests.md, http-status-codes-and-errors.md, http-headers.md,
│   ├── json-guidelines.md, data-formats.md, pagination.md, performance.md,
│   ├── events.md, hyper-media.md, meta-information.md, tooling.md, ...
│   └── models/  # problem-1.0.1.yaml, headers-1.0.0.yaml (Deprecation/Sunset), money-1.0.0.yaml
└── skills/<name>/SKILL.md         # 15 existing skills
```

**The two adaptations (authoritative, from ADAPTATION.md):** major version is a
URL path segment (`/api/v1/...`, rule #114) and everything is served under an
`/api` base path (#135). Media-type versioning is a violation (#115). Canonical
path: `/api/v1/{resources}`.

**Existing skills** — do not duplicate these:

| Builder (edits the API artifact) | Knowledge (advises only) |
|---|---|
| create-new-api, add-collection, add-resource, add-endpoint, new-version | audit, api-compatibility, api-security, error-responses, http-headers, http-methods-and-status, json-conventions, naming-conventions, pagination, url-design |

## Conventions every new skill MUST follow

Copy the patterns from existing skills (best templates: `skills/add-endpoint/SKILL.md`
for builders, `skills/api-compatibility/SKILL.md` for knowledge skills).

1. **Frontmatter** — exactly these fields:
   ```yaml
   ---
   name: <kebab-case, matches directory name>
   description: <what it does + explicit "Use when ..." trigger phrases>
   type: builder | knowledge
   argument-hint: "<...>"   # builders only, if they take arguments
   ---
   ```
2. **Adaptation notice block** — every skill includes, verbatim, right after the
   intro paragraph (copy from any existing skill):
   > **Adaptation:** URL versioning and `/api` base path apply — see the
   > [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) ...
3. **Mode statement** — builders: "This skill **modifies the API artifact**.
   Produce the edited file(s) and a short summary of every change made."
   Knowledge: "This is a **knowledge** skill — it produces guidance and
   reference material, not edits to an artifact. Advise; do not modify files
   unless asked."
4. **Artifact detection (builders)** — reuse the existing "Step 1 — Detect the
   API artifact" wording: look for an OpenAPI doc (`*.yaml`/`*.yml`/`*.json`
   containing `openapi:`/`swagger:`, common locations `openapi.yaml`,
   `api.yaml`, `api/`, `spec/`, `docs/`), degrade gracefully to code-first
   routes (Spring, FastAPI/Flask, Express/Koa, ASP.NET), and state which
   artifact was detected before editing.
5. **Rule citations** — cite rules inline as `(#NNN)` and end with a
   `## Reference` section listing the `reference/<chapter>.md` files and rule
   numbers used, e.g.
   `- reference/deprecation.md — #187/#189 (spec markers, headers)`.
   Verify every cited `#NNN` actually exists: `grep -r "rule-NNN" reference/`.
6. **Rules are distilled inline** — a skill must be usable without opening the
   reference chapters; the Reference section is for full detail only.
7. **Arguments** — if `$ARGUMENTS` contains `/` or spaces that are meaningful,
   say explicitly how to parse it and that it must not be naively split
   (see add-collection/add-endpoint for wording).

## Work items (priority order)

Implement in this order. Items 1–4 are the core deliverable; 5–9 are
lower-priority and can be done in a later pass.

---

### 1. `modify-schema` (builder) — field-level changes with compatibility enforcement

**Why:** The most common API change is adding/renaming/removing a field.
Existing builders only create new paths; `api-compatibility` advises but never
edits. This closes the gap.

- **Directory:** `skills/modify-schema/`
- **argument-hint:** `"<schema-or-path> <add|remove|rename|retype> <field>[ <new-name-or-type>]"`
  (also accept a free-form description; if `$ARGUMENTS` is empty, ask what to change).
- **Steps to write:**
  1. Detect the artifact (standard Step 1 boilerplate).
  2. Locate the target schema: a named `components/schemas` entry, or resolve
     from a resource path. Determine whether the schema is used for **input**,
     **output**, or **both** (walk requestBody/response usages) — the
     compatibility rules differ per direction (#107).
  3. **Classify the change** against the compatible-extension rules (#106,
     #107, #111, #112):
     - Compatible: add optional input field; add output field; make mandatory
       input field optional; make optional output field mandatory; extend an
       *input* enum; add values to an `x-extensible-enum`.
     - **Breaking:** remove/rename any field; change type or semantics; add a
       mandatory input field or make optional input required; tighten
       validation; extend a closed *output* `enum`.
     - Schemas used for both input and output get the intersection (only
       add-optional is safe).
  4. **If breaking → STOP.** Do not apply it. Tell the user it is breaking,
     name the violated rule, and hand off: prefer a compatible alternative or
     new resource variant (#113), else the `new-version` skill. Offer a
     compatible near-equivalent (e.g. rename = add new field + deprecate old
     via the `deprecate` skill).
  5. If compatible, apply the edit obeying payload conventions: snake_case
     `^[a-z_][a-z_0-9]*$` (#118), plural arrays/singular objects (#120), no
     nullable booleans (#122), null ≡ absent (#123), date/time fields named
     `*_at`/`*_date` with RFC 3339 formats (#169/#235), enums as
     `UPPER_SNAKE_CASE` strings (#240) preferring `x-extensible-enum` (#112),
     money via the common Money object (#173), `readOnly`/`writeOnly` for
     response-only/request-only fields (#252), never
     `additionalProperties: false` (#111).
  6. Bump `info.version` per semver (#116): MINOR for compatible additions,
     PATCH for doc-only edits.
- **Output section:** edited artifact + summary with the compatibility verdict
  and the rules checked.
- **Reference section:** compatibility.md (#106–#113), json-guidelines.md
  (#118/#120/#122/#123/#240/#252), data-formats.md (#169), meta-information.md (#116).

---

### 2. `deprecate` (builder) — mark elements deprecated and wire Deprecation/Sunset

**Why:** Deprecation without a version bump is a standalone, common task.
Knowledge exists (`api-compatibility`, `reference/deprecation.md`) but nothing
edits the spec. `new-version` only deprecates as a side effect of a major bump.

- **Directory:** `skills/deprecate/`
- **argument-hint:** `"<operation|path|field|version> [sunset-date]"` — e.g.
  `GET /api/v1/orders/{id}/history`, `Order.legacy_status`, `v1`.
- **Steps to write:**
  1. Detect the artifact; resolve the target (an operation, a whole path, a
     schema property, or an entire version's path set).
  2. **Spec markers (#187):** set `deprecated: true` on the affected
     operation/parameter/property; extend its `description` with the reason,
     planned sunset date, and migration advice (what replaces it).
  3. **Response headers (#189):** on every affected operation add the
     `Deprecation` (RFC 9745 — `@<unix-timestamp>` or `true`) and `Sunset`
     (RFC 8594 — HTTP-date) response headers. Copy the header schemas from
     `${CLAUDE_PLUGIN_ROOT}/reference/models/headers-1.0.0.yaml`
     (`#/Deprecation`, `#/Sunset`) into the spec's own `components/headers` and
     `$ref` them locally (same copy-in pattern `error-responses` uses for the
     Problem schema — the plugin root is not reachable from the user's repo).
     If multiple elements are deprecated, use the **earliest** timestamp.
  4. If no sunset date was given, ask for one or insert a clearly marked
     placeholder and say so.
  5. **Process reminder (do not skip):** headers alone are insufficient —
     output a short checklist: obtain client consent on the sunset date (#185),
     monitor usage of the deprecated element (#188), clients must not newly
     adopt it (#191), shut down only after consumers migrated.
- **Reference section:** deprecation.md (#185/#187/#188/#189/#191),
  models/headers-1.0.0.yaml.

---

### 3. `fix-compliance` (builder) — apply remediations from an audit

**Why:** `audit` produces findings and explicitly refuses to modify files.
Nothing applies the fixes, and the dangerous part — many fixes are themselves
breaking changes — needs encoded judgment.

- **Directory:** `skills/fix-compliance/`
- **argument-hint:** `"[path-or-spec-glob | audit-report-file]"`
- **Steps to write:**
  1. If `$ARGUMENTS` points at an existing audit report, parse its findings;
     otherwise run the `audit` skill's Step 1–2 checklist inline to produce
     findings first (state that you did).
  2. **Partition every finding** into:
     - **Safe to apply now** — spec-internal fixes invisible to clients:
       missing meta-information (`info.contact`, `x-api-id` #215, `x-audience`
       #219, semver #116), missing `Problem`/`default` error responses (#176),
       undocumented status codes, missing pagination *metadata in the spec* for
       already-object-shaped responses, missing security declarations that the
       implementation already enforces, header definitions, descriptions.
     - **Breaking — do NOT auto-apply:** renaming camelCase→snake_case fields
       (#118), un-bare-ing a top-level array (#110), adding the `/api`/`v1`
       path prefix to a served API (#135/#114), pluralizing collections (#134),
       converting verb URLs to noun sub-resources (#141), changing status-code
       semantics. These change the wire contract.
  3. Apply the safe set, one category at a time, matching existing file style.
  4. For the breaking set, output a **migration plan**: for each finding, the
     compatible route (parallel new field + `deprecate` skill) or the versioned
     route (`new-version` skill), and in which order.
  5. Re-run the audit checklist over the edited artifact and report the
     before/after finding counts.
- **Reference section:** point at the same chapters `audit` cites; explicitly
  cross-reference the `audit`, `deprecate`, `new-version`, and `modify-schema`
  skills.

---

### 4. Shared **verification step** in all builder skills (edit existing files)

**Why:** Every builder ends at "edit + summarize" with no validation of the
result. This is where agent output actually fails (broken YAML, dangling
`$ref`s).

- **Change:** add a `## Verify` section (before `## Output`) to **all builder
  skills**: create-new-api, add-collection, add-resource, add-endpoint,
  new-version, plus new ones from this plan. Content (adapt wording per skill):
  1. Re-read the edited file; confirm it parses (for YAML/JSON run a quick
     parse, e.g. `python -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <file>`).
  2. Confirm every `$ref` added resolves within the document.
  3. If a spec linter is available in the project (`spectral`, `redocly`,
     `swagger-cli`, `openapi-spec-validator` in package.json / dev deps / CI
     config), run it and fix errors it reports on the touched paths. Do not
     install tools unasked; if none is available, say so in the summary.
  4. Spot-check the change against this plugin's audit checklist categories
     touched by the edit (paths, status codes, problem+json, pagination shape).
- Keep it short (~10 lines per skill); reference `reference/tooling.md`.

---

### 5. `async-operations` (knowledge) — long-running, batch/bulk, rate limiting

- **Directory:** `skills/async-operations/`
- **description triggers:** "long-running operation", "202", "job/status
  resource", "batch", "bulk", "rate limit", "429".
- **Content to distill:**
  - **Async processing (#253):** `POST`/`PUT`/`PATCH`/`DELETE` may return
    `202 Accepted` with a status URL (job/status resource pattern); model the
    job as a resource (`/api/v1/report-jobs/{id}`) with a state enum; final
    result via the job resource or `Location`. `Prefer: respond-async` (#181)
    as the opt-in signal.
  - **Idempotent retries (#229/#230/#231):** design POST/PATCH idempotent when
    duplicates matter — conditional `If-Match` key, secondary key in the body
    (preferred, #231), or `Idempotency-Key` header (#230).
  - **Batch/bulk (#152):** model as a POST collection of items; respond `207`
    with per-item status objects; the endpoint may still be atomic — then use a
    single normal status code and document it.
  - **Rate limiting (#153):** return `429` with `Retry-After` (or the
    `X-RateLimit-*` trio); clients back off.
- **Reference:** http-requests.md (#229–#231, #253), http-status-codes-and-errors.md
  (#152, #153), http-headers.md (#181/#230).

---

### 6. `secure-api` builder mode (extend `api-security` or new skill)

**Decision made for you:** extend the existing `skills/api-security/SKILL.md`
rather than adding a 16th skill — the knowledge content is the same rules.

- Change `type: knowledge` framing: keep the rules section, add a
  "## Applying it to a spec (builder mode)" section: when the user asks to
  *secure the API* (not just asks how), detect the artifact and edit it —
  add `components/securitySchemes` (`http`+`bearer` JWT for internal
  service-to-service; `oauth2` only when real flows exist, #104), add
  per-operation `security` requirements with scope names matching #225
  (`<application-id>.<access-mode>` or
  `<application-id>.<resource-name>.<access-mode>`, `uid` pseudo-scope for
  genuinely unrestricted endpoints, #105), least-privilege read/write split.
- Update the frontmatter `description` to add builder triggers ("add auth to
  my spec", "wire up security schemes") and soften the "do not modify files"
  line to "advise unless asked to apply".
- Add the Verify section from item 4.

---

### 7. `performance` (knowledge) — uncovered `reference/performance.md`

- **Directory:** `skills/performance/`
- **Content to distill:** reduce bandwidth (#155); gzip with `Accept-Encoding`
  /`Content-Encoding` (#156); partial responses via `fields` query param with
  the field-spec mini-grammar (#157); embedding sub-resources via `embed`
  (#158); document cacheability — default is *not cacheable*
  (`Cache-Control: no-store` semantics); when caching, document `Cache-Control`
  /`Vary`/`ETag` on the cacheable `GET`/`HEAD`/`POST` endpoints (#227) and
  prefer ETag validation (cross-link the http-headers skill for
  `ETag`/`If-Match` mechanics, #182).
- **Reference:** performance.md (#155–#158, #227), http-headers.md (#182).

---

### 8. Extend `json-conventions` with the missing data-format rules

Edit `skills/json-conventions/SKILL.md`, add a "## Data formats" subsection
(keep existing content untouched):

- Standard types/formats table per #238/#171: `integer` needs `format:
  int32|int64|bigint`, `number` needs `format: float|double|decimal`.
- Binary as `base64url` (#239).
- Durations/intervals as ISO 8601 strings (#127).
- Country `ISO 3166-1-alpha-2`, language `BCP 47`, currency `ISO 4217` (#170).
- UUID caution — only when necessary; prefer decision-tree from #144.
- Update the skill's Reference section to add `reference/data-formats.md`
  rule numbers.

---

### 9. `event-design` (knowledge) — OPTIONAL, confirm scope first

`reference/events.md` is the largest chapter (935 lines, rules #194–#214,
#242, #245–#247) with zero skill coverage. **Before building it, ask the
maintainer whether event publishing is in scope for the plugin's users.** If
no: propose removing `reference/events.md` instead. If yes:

- **Directory:** `skills/event-design/`
- Distill: events are part of the service interface (#194); schemas published
  and reviewed (#195–#197); event type naming (#213); ownership (#207); two
  categories — general events vs data-change events (#198, #201, #202);
  mandatory metadata + unique ids (#247, #211); ordering (#203, #242) and hash
  partitioning (#204); no sensitive data (#200); consumers robust to
  duplicates/out-of-order (#214, #212); semantic versioning + compatibility
  mode (#246, #245, #209); avoid `additionalProperties` (#210).

---

## Housekeeping (do last)

1. **ADAPTATION.md maintenance note:** it warns that rule summaries are
   duplicated across builder skills. After adding skills, extend the grep-list
   sentence to include the new skill names (`modify-schema`, `deprecate`,
   `fix-compliance`).
2. **plugin.json:** bump `version` to `1.1.0`.
3. **Cross-references:** in `audit`'s output section, mention that
   `fix-compliance` can apply the findings. In `api-compatibility`'s output,
   mention `modify-schema` (applies compatible changes) and `deprecate`.
   In `new-version`, mention `deprecate` for the old version's markers.

## Acceptance checklist (run per skill, then overall)

- [ ] `skills/<name>/SKILL.md` exists; frontmatter has `name` (== dir name),
      `description` with "Use when ..." triggers, `type`, and (builders)
      `argument-hint`.
- [ ] Adaptation notice block present verbatim; builder/knowledge mode
      statement present.
- [ ] Every `#NNN` cited resolves: `grep -r "rule-NNN" reference/` hits.
- [ ] Every `reference/<file>.md` and `reference/models/*.yaml` path cited
      exists.
- [ ] Skill is self-contained — the rules needed to act are distilled inline,
      not just cited.
- [ ] Builders: artifact-detection step, breaking-change stop conditions, a
      Verify section, and an Output section listing edited files + summary.
- [ ] No two skills claim the same trigger phrases in `description`
      (compare against the table of existing skills above).
- [ ] All builder skills (old and new) contain the shared Verify section.
- [ ] plugin.json bumped; ADAPTATION.md maintenance note updated.
