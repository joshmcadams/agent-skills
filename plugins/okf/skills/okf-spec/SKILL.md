---
name: okf-spec
description: Core Open Knowledge Format (OKF) specification rules, frontmatter fields, and directory conventions. Use as a reference when designing or answering questions about the OKF format, or configuring a knowledge catalog.
type: knowledge
---
# Open Knowledge Format (OKF) Specification

OKF is a vendor-neutral, open spec for representing knowledge as a directory of markdown files with YAML frontmatter. 

## Key Terminology

- **Bundle** — A directory tree of `.md` files. The unit of distribution.
- **Concept** — One markdown file = one unit of knowledge (table, metric, playbook, API, etc.)
- **Concept ID** — File path within the bundle, minus `.md` suffix (e.g., `tables/users`).
- **Frontmatter** — YAML block between `---` delimiters at file top.
- **Body** — Everything after the frontmatter. Standard markdown.

## Frontmatter Fields

| Field | Required? | Description |
|-------|-----------|-------------|
| `type` | **YES** | Kind of concept (e.g. `BigQuery Table`, `Metric`, `Playbook`) |
| `title` | Recommended | Human-readable display name |
| `description` | Recommended | One-sentence summary |
| `resource` | Recommended | URI identifying the underlying asset |
| `tags` | Optional | YAML list for cross-cutting categorization |
| `timestamp` | Optional | ISO 8601 datetime of last meaningful change |

*Additional keys are permitted.*

## Reserved Filenames

- `index.md`: Directory listing for progressive disclosure. *No frontmatter allowed* except at the root level where it may declare `okf_version: "0.1"`.
- `log.md`: Change history, newest first. *No frontmatter allowed*.

## Conventional Body Headings

- `# Schema`: Data assets — describe columns/fields.
- `# Examples`: Show concrete usage (code blocks, queries).
- `# Citations`: List external sources backing claims (numbered).

## Guardrails
- **Minimally opinionated** — Only `type` is required.
- **Format, not platform** — No cloud or SDK dependency.
- **Do not impose taxonomy** — Type values are free-form strings.
- **Broken links permitted** — These are intentionally allowed to point to future concepts.

For the full detailed spec, see `${CLAUDE_PLUGIN_ROOT}/reference/spec-v01.md`.
