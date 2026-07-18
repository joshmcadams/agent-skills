---
name: audit
description: Validate a directory as an OKF bundle. Checks for YAML frontmatter and required 'type' field using okflint or the built-in validation script. Use when the user asks to validate an OKF bundle.
type: named-operation
argument-hint: "[bundle-directory]"
---
# Audit an OKF Bundle

Validate that a directory conforms to the Open Knowledge Format (OKF) specification.

## Conformance Rules

1. Every non-reserved `.md` file has parseable YAML frontmatter
2. Every frontmatter has a non-empty `type` field
3. Reserved files (`index.md`, `log.md`) follow their defined structure when present

## Validation Steps

### 1. Preferred: okflint (when available)

Check if okflint is installed (`command -v okflint`). If NOT installed, ask the user if they'd like to install it (`uv tool install okflint` or `pip install okflint`).

If installed (or user agreed to install):
```bash
if [ -f okf-base.yaml ]; then
  okflint validate --manifest okf-base.yaml ./bundle/
else
  okflint validate ./bundle/
fi
```

### 2. Fallback: built-in bash script

If okflint is not available, use the built-in validation script located at `${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh`.

```bash
${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh ./bundle/
```

### 3. Report Results

Report the results of the validation to the user.
Note that the spec explicitly permits missing optional fields, unknown type values, unknown frontmatter keys, broken links, or missing index files. These are considered Warnings (W), not Errors (E).
