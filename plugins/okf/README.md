# Open Knowledge Format (OKF) Plugin

Skills for creating, validating, and enriching **Open Knowledge Format (OKF)** bundles — the open specification for representing organizational knowledge as markdown files with YAML frontmatter.

Adapted from the official [Google Cloud Knowledge Catalog](https://github.com/GoogleCloudPlatform/knowledge-catalog) and the [fabricioctelles/skills](https://github.com/fabricioctelles/skills) repository.

## Provided Skills

| Kind | Skill | Purpose |
|------|-------|---------|
| Named op | `/audit` | Validate a directory as an OKF bundle (checking frontmatter and structure). |
| Builder | `create-bundle` | Scaffold a new OKF bundle structure. |
| Builder | `add-concept` | Add a new OKF concept (markdown file with frontmatter) to an existing bundle. |
| Builder | `enrich-concept` | Enrich an existing OKF concept with schema, examples, and citations. |
| Knowledge | `okf-spec` | Core OKF format info: frontmatter, structure, terminology. |
| Knowledge | `okf-conversion` | Guide to converting from Notion, Obsidian, or CSV to OKF. |

## Licensing

Licensed under Apache 2.0. See [LICENSE](LICENSE).
