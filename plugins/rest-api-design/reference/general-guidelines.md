# General guidelines

The titles are marked with the corresponding labels: **MUST**, **SHOULD**, **MAY**.


## MUST follow API first principle {#rule-100}

You must follow the *API First Principle*, more specifically:

- You must define APIs first, before coding its implementation, using
  OpenAPI as specification language ([#101](#rule-101))
- You must design your APIs consistently with these guidelines; use our
  [API Linter Service (internal_link)](https://zally.zalando.net/)
  for automated rule checks.
- You must call for early review feedback from peers and client developers, and apply
  [our lightweight API review process (internal_link)](https://api.docs.zalando.net/howto/request-review/)
  for all component external APIs, i.e. all apis
  with `x-api-audience =/= component-internal` (see [#219](#rule-219)).


## MUST provide API Specifications using OpenAPI {#rule-101}

We use the standard provided by the [OpenAPI Initiative](https://www.openapis.org/)
to define API specifications. API designers are required to provide the API
specification using a single self-contained YAML file for better readability.
We encourage using [OpenAPI 3.1 version](https://swagger.io/specification/),
especially for new APIs. The API specification documents should be subject to version
control using a source code management system, and you must publish them following [#192](#rule-192).

**Hint:** Our API infrastructure does not break, but might not yet fully support
all OpenAPI 3.1 changes (e.g. displaying `examples` in Sunrise), and continues
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
Following our [Zalando Tech Radar (internal_link)](https://techradar.zalando.net/languages/graphql.html), we think
that GraphQL has no major benefits, but a couple of downsides compared to REST
as API technology for general purpose peer-to-peer microservice communication.
However, GraphQL can provide a lot of value for specific target domain problems,
especially backends for frontends (BFF) and mobile clients, where typically
many (domain object) resources from different services are queried and
multiple roundtrip overhead should be avoided due to (mobile or public)
network constraints. Therefore we list both technologies on ADOPT, though
GraphQL only supplements REST for the BFF-specific problem domain.


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

Normally, API specification files must be **self-contained**, i.e. files
should not contain references to local or remote content, e.g. `../fragment.yaml#/element` or
`$ref: 'https://github.com/zalando/zally/blob/master/server/src/main/resources/api/zally-api.yaml#/schemas/LintingRequest'`.
The reason is, that the content referred to is *in general* **not durable** and
**not immutable**. As a consequence, the semantic of an API may change in
unexpected ways. (For example, the second link is already outdated due to code restructuring.)

However, you may use remote references to resources accessible by the following
service URLs:

- `https://infrastructure-api-repository.zalandoapis.com/ (internal_link)` – used
  to refer to user-defined, immutable API specification revisions published via the
  internal API repository.
- `https://opensource.zalando.com/restful-api-guidelines/{model.yaml}` – used
  to refer to guideline-defined re-usable API fragments (see `{model.yaml}` files in
  [restful-api-guidelines/models](https://github.com/zalando/restful-api-guidelines/tree/main/models)
  for details).

**Hint:** The formerly used remote references to the `Problem` API fragment
(aliases `https://opensource.zalando.com/problem/` and
`https://zalando.github.io/problem/`) are deprecated, but still supported for
compatibility ([#176](#rule-176) on how to replace).

As we control these URLs, we ensure that their content is **durable** and
**immutable**. This allows to define API specifications by using fragments
published via these sources, as suggested in [#151](#rule-151).
