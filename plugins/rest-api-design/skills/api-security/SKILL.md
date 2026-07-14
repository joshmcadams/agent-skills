---
name: api-security
description: How to secure a REST API — OAuth 2.0 / bearer access tokens, functional scopes (permissions), scope naming, least privilege, and keeping secrets out of URLs. Use when securing an API, adding auth, defining scopes or permissions, deciding how to protect endpoints, or asked to actually wire up/add security schemes and scopes to a spec ("secure this API", "add auth to my spec").
type: hybrid
---

# Secure a REST API

Guidance for protecting REST API endpoints with authentication and
authorization: OAuth 2.0 / bearer access tokens, functional scopes
(permissions), scope naming, and safe handling of credentials. Apply these
rules when designing or reviewing how an API is secured.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](${CLAUDE_PLUGIN_ROOT}/ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.
> (If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin root is the directory two levels above this SKILL.md.)

This is primarily a **knowledge** skill — advise by default. But when the user
asks you to actually *secure the API* / *add auth* / *wire up scopes* (not just
asks how), switch to **builder mode**: detect the artifact and edit it per
"Applying it to a spec" below, then produce the edited file(s) and a summary.

## Rules

1. **Secure every endpoint (#104).** Every endpoint MUST be protected with
   authentication and authorization. Define protection in the OpenAPI spec using
   the `http`+`bearer` scheme (JWT) or an `oauth2` scheme. Most internal,
   service-to-service APIs use JWT bearer tokens
   (`Authorization: Bearer <token>`, based on OAuth 2.0 / RFC 6750).
   Use full `oauth2` flows only for customer/partner-facing APIs that actually
   support them — do not declare `oauth2` (e.g. `implicit`) if the service only
   implements a plain bearer scheme, as it leaks auth-server details.

2. **Define and assign permissions/scopes (#105).** Endpoints that expose
   protected data MUST require client authorization via scopes. Assign scopes
   per operation with a `security` requirement. If an endpoint genuinely needs
   no specific permission (all data is low-classification, or authorization is
   enforced per-object), make it explicit by assigning the `uid` pseudo-scope
   rather than leaving it unprotected.

3. **Follow the scope naming convention (#225).** Scope names MUST match:
   - `<application-id>.<access-mode>` — standard permission (preferred for most
     cases), e.g. `business-partner-service.read`, `fulfillment-order.write`.
   - `<application-id>.<resource-name>.<access-mode>` — resource-specific
     permission for finer access differentiation, e.g.
     `order-management.sales-order.read`.
   - `uid` — pseudo-permission meaning access is not restricted.

   `<application-id>` and `<resource-name>` match `[a-z][a-z0-9-]*`;
   `<access-mode>` is `read` or `write`. Prefer component-level scopes without a
   resource segment to avoid an explosion of fine-grained permissions.

4. **Least privilege.** Grant each operation the narrowest scope that works —
   `read` for safe reads, `write` for mutations. Do not reuse one broad scope
   across unrelated operations.

5. **Never put secrets in URLs.** Keep credentials, tokens, API keys, passwords,
   and PII out of path segments and query parameters (they get logged, cached,
   and stored in history). Pass credentials in the `Authorization` header. With
   a minimal spec you need not redefine `Authorization` on each operation — it is
   implied by the `security` section.

6. **Use TLS.** Serve all endpoints over HTTPS/TLS so tokens and payloads are
   encrypted in transit.

7. **Expose only what the audience should see (#219).** Classify the API by
   intended audience (`component-internal`, `business-unit-internal`,
   `company-internal`, `external-partner`, `external-public`) via
   `/info/x-audience`, and scope data exposure accordingly — do not surface
   internal-only fields or operations on a wider-audience API.

## Example — securitySchemes + scopes

Declare the scheme once, then apply per operation:

```yaml
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

# Optional: a default requirement applied to the whole API
security:
  - BearerAuth: [ business-partner-service.read ]

paths:
  /api/v1/business-partners/{partner-id}:
    get:
      summary: Retrieves information about a business partner
      security:
        - BearerAuth: [ business-partner-service.read ]
    put:
      summary: Updates a business partner
      security:
        - BearerAuth: [ business-partner-service.write ]
```

## Applying it to a spec (builder mode)

When asked to actually secure an existing API (not just explain how):

1. **Detect the API artifact.** Look first for an OpenAPI document
   (`*.yaml`/`*.yml`/`*.json` containing `openapi:`/`swagger:`; common
   locations `openapi.yaml`, `api.yaml`, `api/`, `spec/`, `docs/`). Degrade
   gracefully to code-first security config (Spring Security, FastAPI
   dependencies, Express middleware, etc.) if none exists. State which
   artifact you detected before editing.
2. **Add `components/securitySchemes`.** `http`+`bearer` with
   `bearerFormat: JWT` for internal/service-to-service APIs (the common
   case, #104); a full `oauth2` scheme only for customer/partner-facing APIs
   that actually implement those flows — never declare `oauth2` just because
   it looks more complete.
3. **Add a `security` requirement to every operation** that currently has
   none (#105). Derive the scope name from the resource/operation being
   protected, following the naming convention in rule 3 above; default to the
   component-level form (`<application-id>.<access-mode>`) unless the audit
   or the user calls for resource-level granularity. Assign `uid` only to
   endpoints you can justify as genuinely unrestricted — never leave an
   endpoint with no `security` entry at all.
4. **Apply least privilege**: `read` on safe methods (`GET`/`HEAD`), `write`
   on mutating ones (`POST`/`PUT`/`PATCH`/`DELETE`). Don't reuse one scope
   across unrelated resources.
5. **Check for secrets in URLs** while you're in the spec — flag (and offer to
   fix) any path/query parameter that looks like it carries a credential,
   token, or PII; move it to the `Authorization` header or the request body.

## Verify

1. Re-read the edited file; confirm it still parses (for YAML/JSON run a
   quick parse, e.g.
   `python -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <file>`).
2. Confirm every operation in the touched paths now has a `security`
   requirement (explicit scope or `uid`) — none left silently unprotected.
3. Confirm every scope name matches `<application-id>.<access-mode>` or
   `<application-id>.<resource-name>.<access-mode>` (#225).
4. If a spec linter is configured in the project, run it and fix what it
   flags on the touched paths.

## Result

Produce a security design (or review) in which: every operation has a
`security` requirement; a `securitySchemes` entry defines the bearer/OAuth 2.0
scheme; scope names follow #225; scopes follow least privilege; and no
credentials or PII appear in paths or query parameters. Flag any endpoint that
is unprotected without an explicit `uid`, any secret in a URL, and any scope
that violates the naming pattern.

## Reference

For full detail, consult the guidelines bundled with this plugin (the
`reference/` directory at the plugin root, e.g.
`${CLAUDE_PLUGIN_ROOT}/reference/security.md`):

- `reference/security.md` — #104 (secure endpoints), #105 (define/assign
  scopes), #225 (scope naming convention)
- `reference/meta-information.md` — #219 (API audience)
- `reference/compatibility.md` — #114/#115 (URL versioning; media-type
  versioning is a violation here)
