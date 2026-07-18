---
name: add-concept
description: Add a new Open Knowledge Format (OKF) concept (a markdown file with YAML frontmatter) to an existing bundle. Use when the user wants to add a new document, topic, table, metric, or entity to their knowledge catalog.
type: builder
argument-hint: "<file-path> <type>"
---
# Add an OKF Concept

Add a new concept to an OKF bundle. One concept = one markdown file.

## 1. Create File and Frontmatter

Every concept must have YAML frontmatter with at least the `type` field.
Recommended fields include `title`, `description`, `resource`, `tags`, and `timestamp`.

```markdown
---
type: Metric
title: Monthly Recurring Revenue
description: Sum of all active subscription revenue normalized to monthly.
tags: [revenue, saas]
timestamp: 2026-06-13T10:00:00Z
---
# Monthly Recurring Revenue (MRR)
...
```

## 2. Structure Content

Use standard OKF conventions for the body:
- `# Schema` for data assets
- `# Examples` for code or concrete usage
- `# Citations` for references backing claims

## 3. Cross-Link

Use absolute bundle-relative links (e.g., `[customers](/tables/customers.md)`) or standard relative links (e.g., `[churn](./churn.md)`). Note: broken links are permitted by the spec.

## 4. Update index.md and log.md

- If appropriate, add a link to the new concept in the directory's `index.md`.
- Optionally, add an entry to the root `log.md` under today's date.
