# Adaptation Notice (authoritative for all skills in this plugin)

This guideline set deviates from the upstream
[Zalando RESTful API Guidelines](https://github.com/zalando/restful-api-guidelines)
in two ways:

1. **URL versioning** — The major version is a **path segment** placed
   immediately after the base path: `v<major>`, e.g. `/api/v1/...`,
   `/api/v2/...`. Minor, backward-compatible changes MUST NOT change the URL.
   See reference rules [#114](reference/compatibility.md#rule-114) and
   [#115](reference/compatibility.md#rule-115).

2. **`/api` base path** — All resources are served under an **`/api`** prefix.
   See reference rule [#135](reference/urls.md#rule-135).

The canonical resource path is **`/api/v1/{resources}`**.

Do **NOT** use media-type / `Accept`-header versioning
(`application/vnd...+json;version=2`, `application/x....+json;version=2`) —
that is a violation here. The media type stays plain `application/json`. See
reference rule [#172](reference/json-guidelines.md#rule-172).

## Path convention for bundled files (settled — do not relitigate)

Skills reference bundled files via **`${CLAUDE_PLUGIN_ROOT}/...`**, which
Claude Code substitutes at runtime. For any other agent runtime, every skill
carries the fallback: the plugin root is the directory two levels above the
SKILL.md. Do **not** swap this for relative paths like `../../reference/...`
— a skill executes with the working directory set to the *user's project*,
not the plugin, so bare relative paths resolve to nothing at runtime (this
was tried in commit `5dab25d` and reverted in `d296cc5`). Skills must also
never emit `${CLAUDE_PLUGIN_ROOT}` paths into user artifacts — copy bundled
schemas/headers into the user's spec and `$ref` them locally.

## Maintenance

The naming/schema/status-code rule summaries are duplicated across the
builder skills (`add-collection`, `add-resource`, `add-endpoint`,
`create-new-api`, `new-version`, `modify-schema`, `deprecate`,
`fix-compliance`) and the fundamentals skills. When changing any shared rule
summary, grep all skills for the old wording and update every copy.
