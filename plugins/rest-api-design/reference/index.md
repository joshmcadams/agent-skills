# REST API Guidelines (Adapted) — Reference

This directory is the full reference material cited by the `rest-api-design`
skills. It is an **adaptation** of the
[Zalando RESTful API Guidelines](https://github.com/zalando/restful-api-guidelines)
(© Zalando SE, CC BY 4.0), converted to Markdown and modified. See the
[`NOTICE`](../NOTICE) and [`LICENSE`](../LICENSE) at the plugin root for the
attribution and the full list of changes.

## Adaptation summary

Two rules deviate from upstream Zalando to match common industry practice; these
are the authoritative rules for the skills in this plugin:

- **URL versioning** — major version in the path (`/v1/`), not in a media type
  header. See [compatibility.md](compatibility.md) rules `#114`/`#115`.
- **`/api` base path** — resources are served under an `/api` prefix. See
  [urls.md](urls.md) rule `#135`.

Canonical resource path: `/api/v1/{resources}`.

Rule headings carry anchors like `{#rule-114}`; cross-references appear as `#114`.

## Chapters

- [Introduction](introduction.md)
- [Principles — Design Principles](design-principles.md)
- [General Guidelines](general-guidelines.md)
- [Meta Information](meta-information.md)
- [Security](security.md)
- [REST Basics — Data formats](data-formats.md)
- [REST Basics — URLs](urls.md)
- [REST Basics — JSON payload](json-guidelines.md)
- [REST Basics — HTTP requests](http-requests.md)
- [REST Basics — HTTP status codes and errors](http-status-codes-and-errors.md)
- [REST Basics — HTTP headers](http-headers.md)
- [REST Design — Hypermedia](hyper-media.md)
- [REST Design — Performance](performance.md)
- [REST Design — Pagination](pagination.md)
- [REST Design — Compatibility](compatibility.md)
- [REST Design — Deprecation](deprecation.md)
- [REST Design — API Operation](api-operation.md)
- [REST Design — Events](events.md)
- [Tooling](tooling.md)
- [Best Practices](best-practices.md)
- [References](references.md)
- [Changelog (upstream)](changelog.md)
- [Terms and Definitions](terms.md)

Machine-readable schemas referenced by the guidelines live in [`models/`](models/).
