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

## Maintenance

The naming/schema/status-code rule summaries are duplicated across the
builder skills (`add-collection`, `add-resource`, `add-endpoint`,
`create-new-api`) and the fundamentals skills. When changing any shared rule
summary, grep all skills for the old wording and update every copy.
