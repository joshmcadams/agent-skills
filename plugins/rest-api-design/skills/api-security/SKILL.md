---
name: api-security
description: How to secure a REST API — OAuth 2.0 / bearer access tokens, functional scopes (permissions), scope naming, least privilege, and keeping secrets out of URLs. Use when securing an API, adding auth, defining scopes or permissions, or deciding how to protect endpoints.
---

# Secure a REST API

Guidance for protecting REST API endpoints with authentication and
authorization: OAuth 2.0 / bearer access tokens, functional scopes
(permissions), scope naming, and safe handling of credentials. Apply these
rules when designing or reviewing how an API is secured.

> **Adaptation:** URL versioning and `/api` base path apply — see the
> [shared adaptation notice](../../ADAPTATION.md) for the two deviations
> from upstream Zalando that are authoritative for all skills in this plugin.

## Rules

1. **Secure every endpoint (#104).** Every endpoint MUST be protected with
   authentication and authorization. Define protection in the OpenAPI spec using
   the `http`+`bearer` scheme (JWT) or an `oauth2` scheme. Most internal,
   service-to-service APIs use JWT bearer tokens from the platform IAM token
   service (`Authorization: Bearer <token>`, based on OAuth 2.0 / RFC 6750).
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
`../../reference/security.md`):

- `reference/security.md` — #104 (secure endpoints), #105 (define/assign
  scopes), #225 (scope naming convention)
- `reference/meta-information.md` — #219 (API audience)
- `reference/compatibility.md` — #114/#115 (URL versioning; media-type
  versioning is a violation here)
