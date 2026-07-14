# REST Design - Hypermedia

## MUST use REST maturity level 2 {#rule-162}

We strive for a good implementation of
[REST Maturity Level 2](http://martinfowler.com/articles/richardsonMaturityModel.html#level2) as it enables us to build resource-oriented APIs that
make full use of HTTP verbs and status codes. You can see this expressed
by many rules throughout these guidelines, e.g.:

- [#138](urls.md#rule-138)
- [#141](urls.md#rule-141)
- [#148](http-requests.md#rule-148)
- [#150](http-status-codes-and-errors.md#rule-150)

Although this is not HATEOAS, it should not prevent you from designing
proper link relationships in your APIs as stated in rules below.


## MAY use REST maturity level 3 - HATEOAS {#rule-163}

We do not generally recommend to implement
[REST Maturity Level 3](http://martinfowler.com/articles/richardsonMaturityModel.html#level3). HATEOAS comes with additional API complexity without
real value in our SOA context where client and server interact via REST
APIs and provide complex business functions as part of our e-commerce
SaaS platform.

Our major concerns regarding the promised advantages of HATEOAS (see
also
[RESTistential Crisis over Hypermedia APIs](https://www.infoq.com/news/2014/03/rest-at-odds-with-web-apis),
[Why I Hate HATEOAS](https://jeffknupp.com/blog/2014/06/03/why-i-hate-hateoas/) and others for a detailed discussion):

- We follow the API First principle ([#100](general-guidelines.md#rule-100)) with APIs explicitly defined
  outside the code with standard specification language. HATEOAS does not
  really add value for SOA client engineers in terms of API
  self-descriptiveness: a client engineer finds necessary links and usage
  description (depending on resource state) in the API reference definition
  anyway.
- Generic HATEOAS clients which need no prior knowledge about APIs and
  explore API capabilities based on hypermedia information provided, is a
  theoretical concept that we haven't seen working in practice and does not
  fit to our SOA set-up. The OpenAPI description format (and tooling based
  on OpenAPI) doesn't provide sufficient support for HATEOAS either.
- In practice relevant HATEOAS approximations (e.g. following specifications
  like HAL or JSON API) support API navigation by abstracting from URL
  endpoint and HTTP method aspects via link types. So, Hypermedia does not
  prevent clients from required manual changes when domain model changes
  over time.
- Hypermedia make sense for humans, less for SOA machine clients. We would
  expect use cases where it may provide value more likely in the frontend and
  human facing service domain.
- Hypermedia does not prevent API clients to implement shortcuts and directly
  target resources without 'discovering' them.

However, we do not forbid HATEOAS; you could use it, if you checked its
limitations and still see clear value for your usage scenario that justifies
its additional complexity. If you use HATEOAS please share experience and
present your findings with your API governance group.


## MUST use common hypertext controls {#rule-164}

When embedding links to other resources into representations you must use the
common hypertext control object. It contains at least one attribute:

- `href`: The URI of the resource the hypertext control is linking to.
  All our API are using HTTP(s) as URI scheme.

In API that contain any hypertext controls, the attribute name `href` is
reserved for usage within hypertext controls.

The schema for hypertext controls can be derived from this model:

```yaml
HttpLink:
  description: A base type of objects representing links to resources.
  type: object
  properties:
    href:
      description: Any URI that is using http or https protocol
      type: string
      format: uri
  required:
    - href
```

The name of an attribute holding such a `HttpLink` object specifies the
relation between the object that contains the link and the linked resource.
Implementations should use names from the [IANA Link Relation Registry](http://www.iana.org/assignments/link-relations)
whenever appropriate. As IANA link relation
names use hyphen-case notation, while this guide enforces snake_case
notation for attribute names, hyphens in IANA names have to be replaced
with underscores (e.g. the IANA link relation type `version-history`
would become the attribute `version_history`)

Specific link objects may extend the basic link type with additional
attributes, to give additional information related to the linked
resource or the relationship between the source resource and the linked
one.

E.g. a service providing "Person" resources could model a person who is
married with some other person with a hypertext control that contains
attributes which describe the other person (`id`, `name`) but also the
relationship "spouse" between the two persons (`since`):

```json
{
  "id": "446f9876-e89b-12d3-a456-426655440000",
  "name": "Peter Mustermann",
  "spouse": {
    "href": "https://...",
    "since": "1996-12-19",
    "id": "123e4567-e89b-12d3-a456-426655440000",
    "name": "Linda Mustermann"
  }
}
```

Hypertext controls are allowed anywhere within a JSON model. While this
specification would allow
[HAL](http://stateless.co/hal_specification.html), we actually don't
recommend/enforce the usage of HAL anymore as the structural separation
of meta-data and data creates more harm than value to the
understandability and usability of an API.


## SHOULD use simple hypertext controls for pagination and self-references {#rule-165}

For pagination and self-references a simplified form of the extensible
common hypertext controls ([#164](#rule-164)) should be used to reduce the specification and
cognitive overhead. It consists of a simple URI value in combination with the
corresponding [link relations](http://www.iana.org/assignments/link-relations), e.g. `next`, `prev`, `first`,
`last`, or `self`.

See [#164](#rule-164) and [#161](pagination.md#rule-161) for more information and examples.


## MUST use full, absolute URI for resource identification (with same host, if applicable) {#rule-217}

Links to other resource must always use full, absolute URI.

**Motivation**: Exposing any form of relative URI (no matter if the relative
URI uses an absolute or relative path) introduces avoidable client side
complexity. It also requires clarity on the base URI, which might not be given
when using features like embedding subresources. The primary advantage of
non-absolute URI is reduction of the payload size, which is better achievable
by following the recommendation to use [gzip compression](performance.md#rule-156)

If the linked resource belongs to the same [API](terms.md) i.e. running service, the link should use the same scheme + host + port combination as the original request URI. (This only comes into play when the API is available at e.g. multiple host names.)

**Motivation:** This avoids inconsistent client experience situations, for instance, where authorization is configured for one host
name and not the other, or the second one might not even be accessible from the
network location of the client (like Kubernetes cluster-internal host names).

## MUST not use link headers with JSON entities {#rule-166}

For flexibility and precision, we prefer links to be directly embedded in the
JSON payload instead of being attached using the uncommon link header syntax.
As a result, the use of the [`Link` Header defined by RFC 8288](https://tools.ietf.org/html/rfc8288#section-3) in conjunction with JSON media types is forbidden.
