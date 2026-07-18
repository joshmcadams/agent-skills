---
name: okf-conversion
description: Guide to converting external documentation formats (Notion, Obsidian, CSV, spreadsheets) into OKF bundles. Use when migrating or exporting docs into Open Knowledge Format.
type: knowledge
---
# Convert Sources to OKF

Guidelines for converting external systems into Open Knowledge Format bundles.

For detailed conversion guides, see `${CLAUDE_PLUGIN_ROOT}/reference/conversion.md`.

## Notion Export

1. Properties → YAML frontmatter.
2. Remove Notion UUID suffixes from exported filenames.
3. Convert Notion internal links to standard relative markdown links.
4. Ensure every file has a `type` in the frontmatter.

## Obsidian Vault

1. Convert `[[wikilinks]]` → standard markdown links: `[title](./file.md)`. (Or use okflint which can resolve wikilinks).
2. Move inline `#tags` to the `tags:` array in the YAML frontmatter.
3. Ensure the `type` field exists in the frontmatter of every note you intend to use as a concept.

## CSV / Spreadsheet

1. Each row becomes one concept (one `.md` file).
2. The first column (e.g. `id` or `filename`) determines the file path.
3. Map remaining columns to frontmatter fields (e.g. `type`, `title`, `description`).
4. Any long-form text columns can be placed in the markdown body.

## Guardrails for Conversion
- Preserve existing structure as much as possible.
- If the source system didn't have a `type` concept, you may need to infer it or apply a default `type: Document`.
