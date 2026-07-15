# Agent Skills — Claude Code Skills Marketplace

A Claude Code [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
of personal coding-standards plugins. It currently contains one plugin,
**`rest-api-design`**, which helps agents **design new REST APIs, audit existing
ones, and safely extend them**, following an adapted version of the
[Zalando RESTful API Guidelines](https://github.com/zalando/restful-api-guidelines).
More plugins may be added over time.

## What's inside

### `rest-api-design`

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
| Fundamentals | `api-first`, `naming-conventions`, `url-design`, `json-conventions`, `http-methods-and-status` | Core design rules (incl. spec-first / OpenAPI requirement). |
| Cross-cutting | `pagination`, `error-responses`, `api-compatibility`, `api-security`, `http-headers`, `async-operations`, `performance`, `event-design` | Concerns that span endpoints. |

*"Kind" above is descriptive, not a strict taxonomy: e.g. `/audit` is listed as
a Named op but is implemented with `type: knowledge` (it produces a report and
never modifies an artifact), unlike the other three named ops which use
`type: builder`.*

The full, adapted guidelines are bundled as Markdown under
[`plugins/rest-api-design/reference/`](plugins/rest-api-design/reference/) and are
cited by the skills.

#### Deviations from the upstream Zalando guidelines

This is a **modified** version. Two rules were intentionally reversed to match
common industry practice:

1. **URL versioning instead of media type versioning.** Versions live in the
   path (`/v1/`), not in the `Accept`/`Content-Type` header.
2. **`/api` base path is used** (Zalando recommends against it).

Combined, the canonical resource path convention here is:

```
/api/v1/{resources}
```

See the plugin's [`NOTICE`](plugins/rest-api-design/NOTICE) file for the complete
list of modifications and the attribution required by the upstream license.

## Installing

```
/plugin marketplace add <this-repo-url-or-path>
/plugin install rest-api-design@my-skills
```

Then invoke named operations like `/rest-api-design:audit` or
`/rest-api-design:add-collection orders/line-items`, or just let the knowledge
skills activate automatically while you design an API.

## License & attribution

This repository uses a dual-license structure:

- **Everything outside `plugins/rest-api-design/`** (including this README and
  the marketplace config) is licensed under the permissive [MIT License](LICENSE).
- **`plugins/rest-api-design/`** is licensed separately under
  [Creative Commons Attribution 4.0 International (CC BY 4.0)](plugins/rest-api-design/LICENSE),
  since its bundled guidelines are derived from the Zalando RESTful API
  Guidelines, © Zalando SE. Modifications are documented in
  [`NOTICE`](plugins/rest-api-design/NOTICE). This project is not affiliated
  with or endorsed by Zalando SE.

Each plugin's `plugin.json` declares its own `license` field; check there before
assuming a plugin follows the repo-wide default.
