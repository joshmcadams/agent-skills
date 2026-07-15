# General guidelines

The titles are marked with the corresponding labels: **MUST**, **SHOULD**, **MAY**.


## MUST follow API first principle {#rule-100}

You must follow the *API First Principle*, more specifically:

- You must define APIs first, before coding its implementation, using
  OpenAPI as specification language ([#101](#rule-101))
- You must design your APIs consistently with these guidelines; use an API
  linter for automated rule checks.
- You must call for early review feedback from peers and client developers, and apply
  a lightweight API review process for all component-external APIs, i.e. all APIs
  with `x-api-audience =/= component-internal` (see [#219](meta-information.md#rule-219)).


## MUST provide API Specifications using OpenAPI {#rule-101}

We use the standard provided by the [OpenAPI Initiative](https://www.openapis.org/)
to define API specifications. API designers are required to provide the API
specification using a single self-contained YAML file for better readability.
We encourage using [OpenAPI 3.1 version](https://swagger.io/specification/),
especially for new APIs. The API specification documents should be subject to version
control using a source code management system, and you must publish them following [#192](api-operation.md#rule-192).

**Hint:** Our API infrastructure does not break, but might not yet fully support
all OpenAPI 3.1 changes (e.g. displaying `examples` in some API tooling), and continues
supporting OpenAPI 3.0 (and 2.0, aka Swagger 2).

**Hint:** The *OpenAPI 3.1 major change* is that Schema Object definitions are now
fully compliant with standard JSON Schema. It is an incompatible change since
OpenAPI-specific overrides and keywords like `example`, `nullable`, `discriminator`, `xml`
are removed from the Schema Object (see
[OpenAPI Blog: Migrating from OpenAPI 3.0 to 3.1](https://www.openapis.org/blog/2021/02/16/migrating-from-openapi-3-0-to-3-1-0)).
Check out the [OpenAPI release page](https://github.com/OAI/OpenAPI-Specification/releases)
for all change details.

**Hint:** You might find the [OpenAPI Map](https://openapi-map.apihandyman.io/)
a helpful tool supporting UI navigation for the OpenAPI 3.0 specification.
(Newer OpenAPI versions are not yet supported.)

**Hint:** We do not yet provide guidelines for [GraphQL](https://graphql.org/)
and focus on resource oriented HTTP/REST API style (and related tooling
and infrastructure support).
We think that GraphQL has no major benefits, but a couple of downsides
compared to REST as API technology for general purpose peer-to-peer
microservice communication. However, GraphQL can provide a lot of value
for specific target domain problems, especially backends for frontends
(BFF) and mobile clients, where typically many (domain object) resources
from different services are queried and multiple roundtrip overhead
should be avoided due to (mobile or public) network constraints.
Therefore both technologies have their place, though GraphQL only
supplements REST for the BFF-specific problem domain.


## SHOULD provide API user manual {#rule-102}

In addition to the API Specification, it is good practice to provide an API
user manual to improve client developer experience, especially of engineers
that are less experienced in using this API. A helpful API user manual
typically describes the following API aspects:

- API scope, purpose, and use cases
- concrete examples of API usage
- edge cases, error situation details, and repair hints
- architecture context and major dependencies - including figures and
sequence flows

The user manual must be published online, e.g. via our documentation hosting
platform service, GHE pages, or specific team web servers. Please do not forget
to include a link to the API user manual into the API specification using the
`#/externalDocs/url` property.


## MUST write APIs using U.S. English {#rule-103}


## MUST only use durable and immutable remote references {#rule-234}

> **Adaptation:** The original Zalando guideline carved out a specific set of
> Zalando-hosted URLs (`https://opensource.zalando.com/restful-api-guidelines/…`)
> as the allowed durable exception, because Zalando controls those URLs and
> guarantees their content. This plugin does not control them, so that concrete
> whitelist is generalized below to "URLs whose content the API owner controls
> and can guarantee durable and immutable." The guideline's own re-usable
> fragments (the `Problem` object, the `Money` object, the standard header
> definitions) are instead **bundled with this plugin** under
> [`reference/models/`](models/) and meant to be copied into your spec rather
> than referenced over the network. See the `NOTICE` file.

API specification files should be **self-contained** by default: they should
not contain references to arbitrary local or remote content, e.g.
`../fragment.yaml#/element` or
`$ref: 'https://github.com/example-org/some-api/blob/main/api/some-api.yaml#/schemas/SomeSchema'`.
The reason is that the content referred to is *in general* **not durable** and
**not immutable**. As a consequence, the semantic of an API may change in
unexpected ways. (For example, the second link is already outdated due to code restructuring.)

The only remote references you may use are to resources whose content **you (the
API owner) control** and can guarantee to be **durable** and **immutable** — a
location under your own governance whose content at a given path never changes
meaning. If you cannot make that guarantee for a reference, inline the fragment
instead.

The re-usable API fragments defined by this guideline are **bundled with this
plugin** under [`reference/models/`](models/) rather than served from a remote
URL. Do **not** `$ref` them over the network: copy the fragment you need (e.g.
the `Problem` object, the `Money` object, or a standard header) into your spec's
own `components/` and `$ref` it locally, keeping the spec self-contained (see
[#176](http-status-codes-and-errors.md#rule-176) for the `Problem` object and
[#151](http-status-codes-and-errors.md#rule-151) for referencing fragments).
