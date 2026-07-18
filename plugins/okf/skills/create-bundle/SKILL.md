---
name: create-bundle
description: Create a new Open Knowledge Format (OKF) bundle from scratch. Scaffolds the directory structure, initial index.md, log.md, and optionally the first concept files. Use when the user wants to start a new knowledge base or OKF directory.
type: builder
argument-hint: "<directory-name>"
---
# Create an OKF Bundle

Scaffold a new OKF bundle from scratch.

## 1. Determine Scope

Ask the user what knowledge they are capturing (e.g., tables, metrics, playbooks, APIs) to determine a logical directory structure.

## 2. Scaffold Core Files

Generate the root structure:

- Create the `[directory-name]` directory if it doesn't exist.
- Generate a root `index.md` (with optional `okf_version` frontmatter)
- Generate a `log.md` (chronological change history)

### `index.md` Example

```markdown
---
okf_version: "0.1"
---
# My Knowledge Bundle
- [Concepts](./concepts/) - Core concepts
```

### `log.md` Example

```markdown
# Update Log

## 2026-06-17
- **Initialization**: Bundle created.
```

## 3. Create Initial Concepts

Use the `add-concept` skill for each logical entity the user wants to add right away. Ensure standard `type` frontmatter is included.

## Output

Present the directory tree and file contents. Confirm: "Bundle is OKF v0.1 conformant ✅"
