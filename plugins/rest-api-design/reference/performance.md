# REST Design - Performance


## SHOULD reduce bandwidth needs and improve responsiveness {#rule-155}

APIs should support techniques for reducing bandwidth based on client needs.
This holds for APIs that (might) have high payloads and/or are used in
high-traffic scenarios like the public Internet and telecommunication networks.
Typical examples are APIs used by mobile web app clients with (often) less
bandwidth connectivity. (Zalando is a 'Mobile First' company, so be mindful of
this point.)

Common techniques include:

- compression of request and response bodies (see [#156](#rule-156))
- querying field filters to retrieve a subset of resource attributes (see
  [#157](#rule-157) below)
- `ETag` and `If-Match`/`If-None-Match` headers to avoid re-fetching of
  unchanged resources (see [#182](#rule-182))
- `Prefer` header with `return=minimal` or `respond-async` to anticipate reduced
  processing requirements of clients (see [#181](#rule-181))
- the *Pagination* section for incremental access of larger collections of data items
- caching of master data items, i.e. resources that change rarely or not
  at all after creation (see [#227](#rule-227)).

Each of these items is described in greater detail below.


## SHOULD use `gzip` compression {#rule-156}

A servers and clients should support `gzip` content encoding to reduce the data
transported over the network and thereby speed up response times, unless there
is a good reason against this. Good reasons against compression are:

1. The content is already compressed, or
2. The server has not enough resources to support compression.

While `gzip` content encoding should be the default, servers must also support
unencoded content for testing. This is ensured by content negotiation using the
`Accept-Encoding` request header (see [RFC 9110
Section 12.5.3](https://tools.ietf.org/html/rfc9110#section-12.5.3)). Successful compression is signaled via the `Content-Encoding`
response header (see [RFC 9110 Section 8.4](https://tools.ietf.org/html/rfc9110#section-8.4)). Clients and
servers may support other compression algorithms as `compress`, `deflate`, or
`br`.

To signal server support for compression the API specification should define
both, the `Accept-Encoding` request header and the `Content-Encoding` response
header. This can be done by simply referencing the standard header definitions
as follows (see also the default definition below):

```yaml
components:
  parameters|headers:
    Accept-Encoding:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/Accept-Encoding'
    Content-Encoding:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/Content-Encoding'
```

```yaml
components:
  parameters:
    Accept-Encoding:
        name: Accept-Encoding
        in: header
        required: false
        description: |-
          The **Accept-Encoding** indicates the content encoding (usually a
          compression algorithm) the client understands. The server selects one of
          the proposed encodings and returns his choice in the **Content-Encoding**
          response header.

          Supported compression algorithms of the server are [**gzip**][gzip] and
          **identity**, but other encodings from list below may be support too:

          * **gzip** - a compression format that uses [Lempel-Ziv coding][gzip]
            (LZ77) with a 32-bit CRC.
          * **compress** - a format using the [Lempel-Ziv-Welch][lzw] (LZW)
            algorithm.
          * **deflate** - A compression format that uses the [zlib][zlib] structure
            with the [deflate][deflate] compression algorithm.
          * **br** - a compression format that uses the [Brotli][brotli] algorithm.
          * **identity** - the identity function without modification or compression
            used to indicate unchanged responses. This value is always considered as
            acceptable, even if omitted.

          The server may choose not to compress the body of the response, if the
          identity value is acceptable, e.g. if the content is already compressed
          (e.g. for JPEG content), or if the server is missing resources to perform
          the compression operation.

          **Note:** A client is allowed to explicitly forbid the identity value by
          setting **identity;q=0** or **\*;q=0**, however, the server in return may
          respond with a **406** (Not Acceptable) error.

          See [RFC 9110 Section 12.5.3][rfc-9110-12.5.3] as well as [API Guideline
          Rule #156][api-156] for further details.

          [rfc-9110-12.5.3]: <https://tools.ietf.org/html/rfc9110#section-12.5.3>
          [api-156]: <https://opensource.zalando.com/restful-api-guidelines/#156>
          [gzip]: <https://en.wikipedia.org/wiki/LZ77_and_LZ78#LZ77>
          [lzw]: <https://en.wikipedia.org/wiki/LZW>
          [zlib]: <https://en.wikipedia.org/wiki/Zlib>
          [deflate]: <https://en.wikipedia.org/wiki/DEFLATE>
          [brotli]: <https://en.wikipedia.org/wiki/Brotli>
        schema:
          type: array
          items:
            type: string
        style: simple
        explode: false
        example: [ "gzip", "gzip;q=0.8,deflate;q=0.2", "gzip;q=0.9,identity;q=0" ]
    Content-Encoding:
        required: false
        description: |-
          The **Content-Encoding** header lists any encoding (usually a compression
          algorithm) applied to the message payload - in execution order. This allows
          the recipient to decode the content in order to obtain the original payload
          format.

          Supported compression algorithms of the server are [**gzip**][gzip] and
          **identity**, but other encoding from list below may be support too:

          * **gzip** - a compression format that uses [Lempel-Ziv coding][gzip]
            (LZ77) with a 32-bit CRC.
          * **compress** - a format using the [Lempel-Ziv-Welch][lzw] (LZW)
            algorithm.
          * **deflate** - A compression format that uses the [zlib][zlib] structure
            with the [deflate][deflate] compression algorithm.
          * **br** - a compression format that uses the [Brotli][brotli] algorithm.
          * **identity** - the identity function without modification or compression
            used to indicate unchanged responses. This value is always considered as
            acceptable, even if omitted.

          **Note:** the original media/content type is specified in the
          **Content-Type**-header, while the **Content-Encoding** applies to the
          representation of the data. If the original media is encoded in some way,
          e.g. as a zip file, then this information is not be included in the
          **Content-Encoding** header.

          See [RFC 9110 Section 8.4][rfc-9110-8.4] as well as [API Guideline Rule
          #156][api-156] for further details.

          [rfc-9110-8.4]: <https://tools.ietf.org/html/rfc9110#section-8.4>
          [api-156]: <https://opensource.zalando.com/restful-api-guidelines/#156>
          [gzip]: <https://en.wikipedia.org/wiki/LZ77_and_LZ78#LZ77>
          [lzw]: <https://en.wikipedia.org/wiki/LZW>
          [zlib]: <https://en.wikipedia.org/wiki/Zlib>
          [deflate]: <https://en.wikipedia.org/wiki/DEFLATE>
          [brotli]: <https://en.wikipedia.org/wiki/Brotli>
        schema:
          type: array
          items:
            type: string
        style: simple
        explode: false
        example: [ "gzip", "deflate", "gzip,deflate" ]
```

**Note:** since most server and client frameworks still **do not** support
`gzip`-compression **out-of-the-box** you need to manually activate it, e.g.
[Spring Boot](https://www.callicoder.com/configuring-spring-boot-application/#enabling-gzip-compression-in-spring-boot),
or add a dedicated middleware, e.g. [gin-gionic/gin](https://github.com/gin-contrib/gzip#usage).


## SHOULD support partial responses via filtering {#rule-157}

Depending on your use case and payload size, you can significantly reduce
network bandwidth need by supporting filtering of returned entity fields.
Here, the client can explicitly determine the subset of fields he wants to
receive via the `fields` query parameter. (It is analogue to
[GraphQL `fields`](https://graphql.org/learn/queries/#fields) and simple
queries, and also applied, for instance, for
[Google
Cloud API's partial responses](https://cloud.google.com/storage/docs/json_api/v1/how-tos/performance#partial-response).)


### Unfiltered

```http
GET http://api.example.org/users/123 HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "cddd5e44-dae0-11e5-8c01-63ed66ab2da5",
  "name": "John Doe",
  "address": "1600 Pennsylvania Avenue Northwest, Washington, DC, United States",
  "birthday": "1984-09-13",
  "friends": [ {
    "id": "1fb43648-dae1-11e5-aa01-1fbc3abb1cd0",
    "name": "Jane Doe",
    "address": "1600 Pennsylvania Avenue Northwest, Washington, DC, United States",
    "birthday": "1988-04-07"
  } ]
}
```


### Filtered

```http
GET http://api.example.org/users/123?fields=(name,friends(name)) HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/json

{
  "name": "John Doe",
  "friends": [ {
    "name": "Jane Doe"
  } ]
}
```

The `fields` query parameter determines the fields returned with the response
payload object. For instance, `(name)` returns `users` root object with only
the `name` field, and `(name,friends(name))` returns the `name` and the nested
`friends` object with only its `name` field.

OpenAPI doesn't support you in formally specifying different return object
schemes depending on a parameter. When you define the field parameter, we
recommend to provide the following description: `Endpoint supports filtering
of return object fields as described in
[Rule #157](https://opensource.zalando.com/restful-api-guidelines/#157)`

The syntax of the query `fields` value is defined by the following
[BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form) grammar.

```bnf
<fields>            ::= [ <negation> ] <fields_struct>
<fields_struct>     ::= "(" <field_items> ")"
<field_items>       ::= <field> [ "," <field_items> ]
<field>             ::= <field_name> | <fields_substruct>
<fields_substruct>  ::= <field_name> <fields_struct>
<field_name>        ::= <dash_letter_digit> [ <field_name> ]
<dash_letter_digit> ::= <dash> | <letter> | <digit>
<dash>              ::= "-" | "_"
<letter>            ::= "A" | ... | "Z" | "a" | ... | "z"
<digit>             ::= "0" | ... | "9"
<negation>          ::= "!"
```

**Note:** Following the
[principle of
least astonishment](https://en.wikipedia.org/wiki/Principle_of_least_astonishment), you should not define the `fields` query parameter using
a default value, as the result is counter-intuitive and very likely not
anticipated by the consumer.


## SHOULD allow optional embedding of sub-resources {#rule-158}

Embedding related resources (also know as *Resource expansion*) is a
great way to reduce the number of requests. In cases where clients know
upfront that they need some related resources they can instruct the
server to prefetch that data eagerly. Whether this is optimized on the
server, e.g. a database join, or done in a generic way, e.g. an HTTP
proxy that transparently embeds resources, is up to the implementation.

See [#137](#rule-137) for naming, e.g. "embed" for steering of embedded
resource expansion. Please use the
[BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form) grammar, as
already defined above for filtering, when it comes to an embedding query
syntax.

Embedding a sub-resource can possibly look like this where an order
resource has its order items as sub-resource (/order/{orderId}/items):

```http
GET /order/123?embed=(items) HTTP/1.1

{
  "id": "123",
  "_embedded": {
    "items": [
      {
        "position": 1,
        "sku": "1234-ABCD-7890",
        "price": {
          "amount": 71.99,
          "currency": "EUR"
        }
      }
    ]
  }
}
```


## MUST document cacheable `GET`, `HEAD`, and `POST` endpoints {#rule-227}

Caching has to take many aspects into account, e.g. general *cacheability* of response information, our guideline to protect endpoints
using SSL and OAuth authorization ([#104](#rule-104)), resource update and invalidation
rules, existence of multiple consumer instances. Caching is in best case
complex, e.g. with respect to consistency, in worst case inefficient.

As a consequence, client side as well as transparent web caching should be
avoided, unless the service supports and requires it to protect itself, e.g.
in case of a heavily used and therefore rate limited master data service, i.e.
data items that rarely or not at all change after creation.

As default, servers and clients should always set the `Cache-Control` header
to `Cache-Control: no-store` and assume the same setting, if no `Cache-Control`
header is provided.

**Note:** There is no need to document this default setting. However, please make
sure that your framework is attaching these header values by default, or ensure
this manually, e.g. using the best practice of
[Spring Security](https://www.baeldung.com/spring-security-cache-control-headers)
as shown below. Any setup deviating from this default must be sufficiently
documented.

```http
Cache-Control: no-cache, no-store, must-revalidate, max-age=0
```

If your service really requires to support caching, please observe the
following rules:

- Document all *cacheable* `GET`, `HEAD`, and `POST` endpoints by declaring
  the support of `Cache-Control`, `Vary`, and `ETag` headers in response.
  **Note:** you must not define the `Expires` header to prevent redundant and
  ambiguous definition of cache lifetime. A sensible default documentation of
  these headers is given below.
- Take care to specify the ability to support caching by defining the right
  caching boundaries, i.e. time-to-live and cache constraints, by providing
  sensible values for `Cache-Control` and `Vary` in your service. We will
  explain best practices below.
- Provide efficient methods to warm up and update caches (see the *Cache Support Patterns* section).

Usually, you can reuse the standard `Cache-Control`, `Vary`, and `ETag`
response header definitions provided by the guideline as follows:

```yaml
components:
  parameters|headers:
    Cache-Control:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/Cache-Control'
    Vary:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/Vary'
    ETag:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/ETag'
```

See also [#182](#rule-182).

### Cache Support Patterns

To make best use of caching in micro service environments you need to provide
efficient methods to warm up and update caches, e.g. as follows:

- In general, you should support `ETag` Together With `If-Match`/`If-None-Match` Header ([#182](#rule-182)) on all *cacheable* endpoints.
- For larger data items you should support `HEAD` requests or more a bit more
  efficient `GET` requests with `If-None-Match` header to check for updates.
- For small data sets you should provide a **get full collection** `GET` endpoint
  that supports an `ETag` for the collection in combination with a `HEAD` or
  `GET` requests with `If-None-Match` to check for updates.
- For medium sized data sets provide a **get full collection** `GET` endpoint
  that supports an `ETag` for the collection in combination with the *Pagination* section
  and `<entity-tag>` filtering `GET` requests for limiting the response to
  changes since the provided `<entity-tag>`. **Note:** this is not supported by
  generic client and proxy caches on HTTP layer.

**Hint:** For proper cache support, you must return `304` without content on a
failed `HEAD` or `GET` request with `If-None-Match: <entity-tag>` ([#182](#rule-182))
instead of `412`. For `ETag` see also [#182](#rule-182).

#### Default Header Values

The default setting for `Cache-Control` should contain the `private` directive
for endpoints with standard OAuth authorization ([#104](#rule-104)), as well as the
`must-revalidate` directive to ensure, that the client does not use stale cache
entries. Last, the `max-age` directive should be set to a value between a few
seconds (`max-age=60`) and a few hours (`max-age=86400`) depending on the change
rate of your master data and your requirements to keep clients consistent.

```http
Cache-Control: private, must-revalidate, max-age=300
```

The default setting for `Vary` is harder to determine correctly. It depends on
the API endpoint, e.g. whether it supports compression, accepts different media
types, or requires other request specific headers. To support correct caching
you have to carefully choose the value. However, a good first default may be:

```http
Vary: accept, accept-encoding
```

Anyhow, this is only relevant, if you encourage clients to install generic
HTTP layer client and proxy caches.

#### Caching strategy

Generic client and proxy caching on HTTP level is hard to configure. Therefore,
we strongly recommend to attach the (possibly distributed) cache directly to
the service (or gateway) layer of your application. This relieves the service
from interpreting the `Vary` header and greatly simplifies the usage patterns
of the `Cache-Control` and `ETag` headers. Moreover, is highly efficient with
respect to cache performance and overhead, and allows to support more
advanced cache update and warm up patterns (see the *Cache Support Patterns* section).

Anyhow, please carefully read [RFC 9111](https://tools.ietf.org/html/rfc9111) before adding any client or
proxy cache.
