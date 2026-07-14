# REST APIs — Claude Code Skills Marketplace

A Claude Code [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
that helps agents **design new REST APIs, audit existing ones, and safely extend
them**, following an adapted version of the
[Zalando RESTful API Guidelines](https://github.com/zalando/restful-api-guidelines).

## What's inside

One plugin, **`rest-api-design`**, containing:

| Kind | Skill | Purpose |
|------|-------|---------|
| Named op | `/audit` | Audit a repository's API(s) for compliance with these guidelines. |
| Named op | `/add-collection <[parent/]collection>` | Add a new collection resource. |
| Named op | `/add-resource <[collection/]resource>` | Add a new resource. |
| Named op | `/new-version <version>` | Introduce a new major API version. |
| Builder | `add-endpoint` | Add an operation (GET/POST/…) to an existing resource. |
| Builder | `create-new-api` | Bootstrap a brand-new API specification from scratch. |
| Builder | `modify-schema` | Add/remove/rename/retype a field, enforcing compatible-extension rules. |
| Builder | `deprecate` | Mark an operation/path/field/version deprecated and wire `Deprecation`/`Sunset` headers. |
| Builder | `fix-compliance` | Apply audit findings, separating safe spec-internal fixes from breaking ones. |
| Fundamentals | `naming-conventions`, `url-design`, `json-conventions`, `http-methods-and-status` | Core design rules. |
| Cross-cutting | `pagination`, `error-responses`, `api-compatibility`, `api-security`, `http-headers`, `async-operations`, `performance`, `event-design` | Concerns that span endpoints. |

*"Kind" above is descriptive, not a strict taxonomy: e.g. `/audit` is listed as
a Named op but is implemented with `type: knowledge` (it produces a report and
never modifies an artifact), unlike the other three named ops which use
`type: builder`.*

The full, adapted guidelines are bundled as Markdown under
[`plugins/rest-api-design/reference/`](plugins/rest-api-design/reference/) and are
cited by the skills.

## Deviations from the upstream Zalando guidelines

This is a **modified** version. Two rules were intentionally reversed to match
common industry practice:

1. **URL versioning instead of media type versioning.** Versions live in the
   path (`/v1/`), not in the `Accept`/`Content-Type` header.
2. **`/api` base path is used** (Zalando recommends against it).

Combined, the canonical resource path convention here is:

```
/api/v1/{resources}
```

See the [`NOTICE`](NOTICE) file for the complete list of modifications and the
attribution required by the upstream license.

## Installing

```
/plugin marketplace add <this-repo-url-or-path>
/plugin install rest-api-design@rest-apis
```

Then invoke named operations like `/rest-api-design:audit` or
`/rest-api-design:add-collection orders/line-items`, or just let the knowledge
skills activate automatically while you design an API.

## License & attribution

The bundled guidelines are derived from the Zalando RESTful API Guidelines,
© Zalando SE, licensed under
[Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE). This
adaptation is likewise distributed under CC BY 4.0. Modifications are documented
in [`NOTICE`](NOTICE). This project is not affiliated with or endorsed by Zalando SE.
