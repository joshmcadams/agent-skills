---
name: new-version
description: Introduce a new MAJOR version of a REST API using URL versioning, keeping the previous version running in parallel and deprecating it. Use when the user asks to create a new API version, bump the API to v2, or make a breaking API change.
type: builder
argument-hint: "<version>"
---

# Introduce a New Major API Version

Introduce a new **major** version of an existing REST API. Because this plugin
versions via the **URL path**, this means adding a new `/api/v{n}/...` path set
alongside the current one, running both **in parallel**, and deprecating the old
version through the `Deprecation`/`Sunset` headers and the deprecation process.
A new major version is a **last resort** — first confirm the change genuinely
cannot be made compatibly.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](../../ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.

## Arguments

`$ARGUMENTS` is the target major version, e.g. `2` or `v2`. **Normalize both to
`v2`** (strip a leading `v` if present, prepend `v`). The segment is `v` followed
by a positive integer, no leading zeros (`v2`, `v3`, …). If `$ARGUMENTS` is empty,
infer the next major from the current highest version segment in the spec (e.g.
`v1` → `v2`) and state what you chose. If the existing API has **no** version
segment at all, treat the current paths as `v1` and introduce `v2` alongside
them.

## Step 1 — Detect the artifact

1. Find the **OpenAPI document** (`*.yaml`/`*.yml`/`*.json` containing `openapi:`
   or `swagger:`; common names/locations `openapi.yaml`, `api.yaml`, under `api/`,
   `spec/`, `docs/`). This is the primary target.
2. If none exists, fall back to **code-first** routes (Spring
   `@RequestMapping`/`@GetMapping`, FastAPI/Flask decorators, Express routers,
   ASP.NET controllers). State the assumption and apply the change in that idiom
   (e.g. add a `/api/v2` router group beside the existing one).
3. Tell the user which artifact you detected before editing.

## Step 2 — Confirm a new version is actually justified (#113, #106–#108)

**Do not create a version if a compatible change is possible.** Verify the change
is genuinely **incompatible**, then prefer alternatives in this order:

- [ ] Can it be a **compatible extension**? For output: add fields, make optional
  fields mandatory (not vice-versa), never remove fields, never extend output
  `enum` ranges. For input: add optional (never mandatory) fields, never tighten
  validation (#107). If yes — **stop**, make the compatible change, no new
  version (#106, #113).
- [ ] If incompatible, prefer a **new resource variant** or a **new service
  endpoint** over a URL version (#113). Only when neither fits do you introduce a
  new major URL version supported in parallel (#113, #114).
- [ ] Confirm the incompatibility is real (changed field semantics, removed
  fields, tightened constraints, restructured payloads) and record why a
  compatible path is impossible. State this reasoning to the user.

## Step 3 — Add the new version's path set (#114, #135, #134, #129)

Edit the **single** OpenAPI document so both versions coexist in it (this is one
artifact, not a second file):

- [ ] Duplicate the affected paths under the new segment: `/v1/customers` gains a
  sibling `/v2/customers` (effective `/api/v2/customers`). Keep `/api` in
  `servers` (#135); the version stays in the path keys (#114).
- [ ] Only the **major** number goes in the path. Minor/patch/compatible changes
  MUST NOT change the URL (#114).
- [ ] Do **not** version via media type — keep `application/json` (#115, #172).
- [ ] Preserve resource-naming rules in the new paths: plural (#134), kebab-case
  (#129), domain-specific (#142). Apply the breaking schema changes only within
  the `v2` operations/schemas (e.g. `SalesOrderV2` or a versioned component),
  leaving `v1` schemas untouched.
- [ ] All resources of the new version share the `v2` segment; do not mix
  versions of the same API arbitrarily (#114).

## Step 4 — Keep the previous version running in parallel (#113, #114)

- [ ] Leave every `v1` path, operation, and schema fully operational. Introducing
  `v2` must not remove or break `v1` — removal is a separate, later step gated on
  consumer consent (#185).
- [ ] Both `/api/v1/...` and `/api/v2/...` are served simultaneously during the
  migration window.

## Step 5 — Deprecate the previous version (#187, #189, #185)

- [ ] **In the spec (#187):** mark the old version's operations/schemas
  `deprecated: true` and add a `description` explaining the deprecation, what to
  use instead (the `v2` equivalents), and how to migrate. If a shutdown is
  planned, state the sunset date.
- [ ] **In responses (#189):** add a `Deprecation` header and, if a shutdown is
  planned, a `Sunset` header to the deprecated `v1` responses. Note the different
  formats:
  ```txt
  Deprecation: @1758095283                    # RFC 9745 — timestamp, or `true`
  Sunset: Wed, 31 Dec 2025 23:59:59 GMT        # RFC 8594 — HTTP-date
  ```
  Define them as reusable response headers in `components` (see
  `reference/models/headers-1.0.0.yaml`). Do **not** use `Link rel="sunset"` or
  the `Warning` header (#189).
- [ ] Headers alone do **not** grant permission to shut down: obtain client
  consent on a sunset date (#185) and monitor usage of the deprecated version
  (#188) before removing it. Clients must not on-board onto deprecated `v1`
  (#191).

## Step 6 — Update meta-information (#116)

- [ ] Bump `info.version` (semver) to reflect the incompatible change, e.g.
  `1.x.y` → **`2.0.0`** (#116). This is the **spec document version** and is
  distinct from the URL segment: the path carries only the **major** (`v2`),
  while `info.version` is full `MAJOR.MINOR.PATCH`. Minor/patch never appear in
  the path.
- [ ] Review `servers` — keep the `/api` base path; the URL version is in the
  path, so `servers` normally does not change. Adjust only if deployment
  routing genuinely differs.

## Step 7 — Communicate migration (#185, #186, #189)

- [ ] Provide a migration summary: what changed and why it is incompatible, the
  `v1`→`v2` endpoint mapping, the deprecation timestamp and (if set) sunset date,
  and required consumer actions.
- [ ] For external partners, respect the agreed after-deprecation life span
  (#186) and collect consent (#185) before any shutdown.

## Output / result

- The **edited OpenAPI artifact** now containing **both** versions: untouched
  `v1` paths (marked `deprecated`) and the new `v2` path set, plus the
  `Deprecation`/`Sunset` response headers and bumped `info.version`.
- A **deprecation & migration summary** (Markdown): version bump, parallel-run
  window, `v1`→`v2` mapping, deprecation/sunset dates, and consumer actions.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`../../reference/<file>.md`):

- `reference/compatibility.md` — #113 (avoid versioning), #106–#108 (compatible
  changes), #114 (URL versioning), #115 (no media-type versioning), #116 usage
- `reference/deprecation.md` — #187 (spec), #189 (`Deprecation`/`Sunset` headers),
  #185/#186/#188/#191 (consent, monitoring, process)
- `reference/urls.md` — #135 (`/api` base path), #134/#129/#142 (naming)
- `reference/meta-information.md` — #116 (semantic versioning of `info.version`)
- `reference/models/headers-1.0.0.yaml` — reusable `Deprecation`/`Sunset` headers
- `reference/index.md` — chapter map and adaptation summary
