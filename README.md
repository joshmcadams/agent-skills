# Agent Skills — Claude Code Skills Marketplace

A Claude Code [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
of personal coding-standards plugins:

- **`rest-api-design`** — **design new REST APIs, audit existing ones, and safely
  extend them**, following an adapted version of the
  [Zalando RESTful API Guidelines](https://github.com/zalando/restful-api-guidelines).
- **`go-ddd`** — **build and audit Domain-Driven Go backend services** (onion
  architecture, CQRS, transactional outbox, idempotent commands), following an
  adapted version of the [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd)
  patterns.
- **`okf`** — **create, validate, and enrich Open Knowledge Format (OKF) bundles**,
  adapted from the official [Google Knowledge Catalog](https://github.com/GoogleCloudPlatform/knowledge-catalog) and [fabricioctelles/skills](https://github.com/fabricioctelles/skills).
- **`mattpocock-skills`** — **AI skills for engineering and productivity**,
  adapted from [Matt Pocock's skills repository](https://github.com/mattpocock/skills).

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

### `go-ddd`

Skills for building and auditing **Domain-Driven Go backend services** — the
onion architecture with dependencies pointing inward, entities and value objects
that guard their own invariants, aggregates referenced by Id, repository
interfaces in the domain with sqlc in infrastructure, CQRS, a transactional
outbox for domain events, and race-safe idempotent commands.

| Kind | Skill | Purpose |
|------|-------|---------|
| Named op | `/audit` | Audit a Go service against the go-ddd architecture and conventions (grep-able checklist). |
| Builder | `add-aggregate` | Add a new aggregate end-to-end across all four layers. |
| Builder | `add-value-object` | Add an immutable, value-compared type (Money, Email, …) validated at construction. |
| Builder | `add-command` | Add a write use case (command + result + idempotent service method) to an aggregate. |
| Builder | `add-query` | Add a read use case (query + output result + read repository method). |
| Builder | `add-domain-event` | Add a past-tense event recorded by the aggregate and wired through the outbox. |
| Builder | `add-repository-method` | Add an access pattern: interface method → sqlc query → generated code → impl. |
| Builder | `add-migration` | Add a golang-migrate `up`/`down` pair, then regenerate sqlc. |
| Builder | `modify-schema` | Add/remove/rename/retype a field across every layer coherently. |
| Fundamentals | `architecture`, `entities`, `value-objects`, `aggregates`, `repositories` | The onion + import rules, and the core domain building blocks. |
| Cross-cutting | `cqrs`, `domain-events-outbox`, `idempotency`, `testing`, `conventions` | Concerns that span the layers. |

The adapted guidance is bundled as Markdown under
[`plugins/go-ddd/reference/`](plugins/go-ddd/reference/) (architecture, entities,
value objects, aggregates, repositories, CQRS, domain events & outbox,
idempotency, testing, conventions, tooling) and is cited by the skills.

Unlike `rest-api-design`, this plugin does not adapt an external *standard* — it
encodes one project's opinionated house style. The choices (e.g. `Id` over `ID`,
sqlc over an ORM) are deliberate and documented in the plugin's
[`NOTICE`](plugins/go-ddd/NOTICE); adopt them wholesale or adapt them.

### `okf`

Skills for creating, validating, and enriching **Open Knowledge Format (OKF)** bundles — the open specification for representing organizational knowledge as markdown files with YAML frontmatter.

| Kind | Skill | Purpose |
|------|-------|---------|
| Named op | `/audit` | Validate a directory as an OKF bundle (checking frontmatter and structure). |
| Builder | `create-bundle` | Scaffold a new OKF bundle structure. |
| Builder | `add-concept` | Add a new OKF concept to an existing bundle. |
| Builder | `enrich-concept` | Enrich an existing OKF concept with schema, examples, and citations. |
| Knowledge | `okf-spec` | Core OKF format info: frontmatter, structure, terminology. |
| Knowledge | `okf-conversion` | Guide to converting from Notion, Obsidian, or CSV to OKF. |

### `mattpocock-skills`

A curated selection of skills from Matt Pocock's repository for engineering, codebase design, and productivity.

| Kind | Skill | Purpose |
|------|-------|---------|
| Named op | `design-an-interface` | Generate multiple radically different interface designs for a module. |
| Named op | `grill-with-docs` | A relentless interview to sharpen a plan or design, creating ADRs and glossaries. |
| Named op | `domain-modeling` | Build and sharpen a project's domain model and ubiquitous language. |
| Named op | `code-review` | Review changes along two axes: Standards and Spec. |
| Named op | `codebase-design` | Shared vocabulary for designing deep modules. |
| Named op | `grill-me` | A relentless interview to sharpen a plan or design. |
| Named op | `teach` | Teach the user a new skill or concept. |
| Knowledge | `writing-great-skills` | Reference for writing and editing skills well. |
| Named op | `handoff` | Compact the current conversation into a handoff document for another agent. |

## Installing

```
/plugin marketplace add <this-repo-url-or-path>
/plugin install rest-api-design@my-skills
/plugin install go-ddd@my-skills
/plugin install okf@my-skills
/plugin install mattpocock-skills@my-skills
```

Then invoke named operations like `/rest-api-design:audit` or
`/rest-api-design:add-collection orders/line-items`, or `/go-ddd:audit` and
`/go-ddd:add-aggregate Review`, or `/okf:audit` and `/okf:create-bundle my-knowledge-base`,
or just let the knowledge skills activate automatically.

## License & attribution

This repository uses a mixed-license structure:

- **Everything outside the plugin directories** (including this README and the
  marketplace config) is licensed under the permissive [MIT License](LICENSE).
- **`plugins/rest-api-design/`** is licensed separately under
  [Creative Commons Attribution 4.0 International (CC BY 4.0)](plugins/rest-api-design/LICENSE),
  since its bundled guidelines are derived from the Zalando RESTful API
  Guidelines, © Zalando SE. Modifications are documented in
  [`NOTICE`](plugins/rest-api-design/NOTICE). This project is not affiliated
  with or endorsed by Zalando SE.
- **`plugins/go-ddd/`** is licensed under the [MIT License](plugins/go-ddd/LICENSE),
  since its bundled guidance is derived from the MIT-licensed
  [`sklinkert/go-ddd`](https://github.com/sklinkert/go-ddd) project, © 2023 Simon
  Klinkert. Attribution and the deliberate design choices are documented in
  [`NOTICE`](plugins/go-ddd/NOTICE). This project is not affiliated with or
  endorsed by that project or its author.
- **`plugins/okf/`** is licensed under the [Apache 2.0 License](plugins/okf/LICENSE),
  since its bundled guidance is derived from the Apache 2.0-licensed
  Google Knowledge Catalog and fabricioctelles/skills repositories. Attribution
  is documented in [`NOTICE`](plugins/okf/NOTICE).
- **`plugins/mattpocock-skills/`** is licensed under the [MIT License](plugins/mattpocock-skills/LICENSE),
  originating from [mattpocock/skills](https://github.com/mattpocock/skills), © 2026 Matt Pocock.

Each plugin's `plugin.json` declares its own `license` field; check there before
assuming a plugin follows the repo-wide default.
