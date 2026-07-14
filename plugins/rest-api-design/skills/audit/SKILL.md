---
name: audit
description: Audit a repository's REST API(s) for compliance with the adapted Zalando REST API guidelines (URL versioning, /api base path). Use when the user asks to audit, review, or check an API for guideline compliance, or asks "is this API compliant?".
argument-hint: "[path-or-spec-glob]"
---

# Audit REST API Compliance

Audit the REST API(s) in this repository against the adapted Zalando REST API
Guidelines bundled with this plugin, and produce a prioritized findings report.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](../../ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.

## Arguments

`$ARGUMENTS` (optional) narrows the audit to a path or glob (e.g. `api/openapi.yaml`
or `services/orders`). If empty, audit the whole repository.

## Step 1 — Locate the API definition

1. Look first for **OpenAPI documents**: `*.yaml`/`*.yml`/`*.json` containing
   `openapi:` or `swagger:` (common names/locations: `openapi.yaml`, `api.yaml`,
   `api/`, `spec/`, `docs/`).
2. If none exist, fall back to **code-first** route definitions (Spring
   `@RequestMapping`/`@GetMapping`, FastAPI/Flask decorators, Express/Koa routers,
   ASP.NET controllers, etc.).
3. State clearly which artifact(s) you found and are auditing. If both exist,
   audit the OpenAPI document and note drift from the code.

## Step 2 — Check each category

Report a finding for every deviation. Tag severity by obligation level:
**MUST** = high, **SHOULD** = medium, **MAY**/style = low.

**Base path & versioning (the deviations — check first):**
- [ ] Resource paths are served under an **`/api`** base path (#135). Root-served
  APIs are a finding.
- [ ] A **major version path segment** `v<n>` follows the base path, e.g.
  `/api/v1/...` (#114). Missing version segment is a finding.
- [ ] **No media-type versioning**: reject custom versioned media types like
  `application/vnd.x+json;version=2` or `application/x....+json;version=2` and
  any `Accept`/`Content-Type`-based version negotiation (#115). Media type is
  plain `application/json` (#172).
- [ ] Only the **major** version appears in the path; minor changes do not change
  the URL (#113).

**URLs & naming:**
- [ ] Collection names are **plural** (#134) and **domain-specific** (#142).
- [ ] Path segments are **kebab-case** `^[a-z][a-z\-0-9]*$` (#129).
- [ ] URLs are **verb-free / noun-based** (#141, #138).
- [ ] Sub-resources use path segments `/parents/{id}/children/{id}` (#143);
  nesting depth ≤ 3 (#147); resource types limited (#146).
- [ ] Query parameters are **snake_case** (#130) and use conventional names
  (`q`, `sort`, `fields`, `limit`, `cursor`, `offset`) (#137).
- [ ] Normalized paths: no trailing/duplicate slashes (#136).

**HTTP methods & status codes:**
- [ ] Standard method semantics; safe/idempotent methods respected (#148, #149).
- [ ] Only **standardized status codes** used and documented per operation (#150,
  #151, #243); `207`/`429`/`503` etc. used where relevant.

**JSON payloads:**
- [ ] Top-level responses are **JSON objects**, never arrays/maps (#110).
- [ ] Property names are **snake_case** (#118), consistent, pluralized arrays
  (#120); booleans/nulls used per rules (#122, #123).
- [ ] Standard formats: dates `RFC 3339` (#169, #126), money as decimal string
  (#171), enums as strings (#240).

**Pagination:**
- [ ] Collection endpoints paginate (#159) using cursor- or offset-based schemes;
  responses wrap items in an object with pagination metadata (#161, #248).

**Errors:**
- [ ] Errors use **`application/problem+json`** (#176) per RFC 9457; a consistent
  problem schema is used; no leaking internals.

**Headers:**
- [ ] Standard headers used correctly; custom headers follow conventions (#183);
  `ETag`/`If-Match` for optimistic locking where applicable (#182).

**Security:**
- [ ] Endpoints are protected (OAuth2 scopes / auth) (#104, #105); scope naming
  conventions followed (#225); no secrets in URLs.

**Compatibility & meta:**
- [ ] Changes are backward-compatible where possible (#106–#108); deprecation
  uses `Deprecation`/`Sunset` headers and process (#187, #189).
- [ ] Spec has required meta-information: `info`, contact, audience (#218, #219).

## Step 3 — Produce the report

Output a Markdown report:
1. **Summary**: artifact(s) audited, counts by severity.
2. **Findings** grouped by category, most severe first. Each finding:
   - Rule reference (e.g. `#114`), obligation (MUST/SHOULD/MAY), severity.
   - Location (`file:line` or path/operation).
   - What's wrong and the concrete remediation.
3. **Quick wins** and **larger refactors** called out separately.

Do not modify files during an audit unless the user explicitly asks for fixes.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`../../reference/<file>.md`):

- `reference/urls.md` — #135 (`/api` base path), naming, structure
- `reference/compatibility.md` — #114/#115 (URL versioning), #106–#108, #113
- `reference/http-requests.md`, `reference/http-status-codes-and-errors.md`
- `reference/json-guidelines.md`, `reference/data-formats.md`
- `reference/pagination.md`, `reference/http-headers.md`, `reference/security.md`
- `reference/meta-information.md`, `reference/deprecation.md`
- `reference/index.md` — chapter map and adaptation summary
