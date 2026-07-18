---
name: enrich-concept
description: Enrich an existing OKF concept with schema, examples, and citations. Automatically fill recommended missing frontmatter. Use when asked to improve, enrich, or document an existing markdown file in an OKF bundle.
type: builder
argument-hint: "<file-path>"
---
# Enrich an OKF Concept

Improve an existing OKF concept by adding missing metadata and structured content sections.

## 1. Fill Recommended Frontmatter

Ensure the following exist in the YAML frontmatter. Derive values from the body content if possible:
- `title`
- `description`
- `tags`
- `timestamp`

## 2. Add Standard Sections

### Add Schema Section (For Data Assets)

For tables, APIs, or structured data, add a `# Schema` section with a table:
```markdown
# Schema
| Column | Type | Description |
|--------|------|-------------|
| `order_id` | STRING | Unique identifier |
```

### Add Examples Section

For APIs, queries, or tools, add an `# Examples` section with fenced code blocks.

### Add Citations Section

If claims reference external sources, add `# Citations` at the bottom, numbered:
```markdown
# Citations
[1] [Official docs](https://example.com/docs)
```

## 3. Weave Cross-Links

Weave absolute or relative cross-links into the natural prose based on discovered relationships (e.g. foreign keys, shared tags).

## 4. Guardrails

1. **NEVER invent data.** Do not fabricate URLs, column names, or types.
2. **Preserve unknown fields.** OKF allows extension. Do not delete unrecognized fields.
3. **Broken links are OK.** They represent not-yet-written knowledge.
