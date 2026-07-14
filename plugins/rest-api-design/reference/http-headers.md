# REST Basics - HTTP headers

In this section we explain a handful of (standard) HTTP headers, which we found
most useful in certain situations and that require additional explanation to be
used effectively.

**Note:** we generally discourage the usage of proprietary headers. However,
they are sometimes useful to pass generic, service independent, overarching
information relevant for our specific application architecture. We consistently
define these proprietary headers below.

Whether a service supports a certain header is part of the service contract.
Therefore APIs should use the `parameters` and `headers` definition of the API
endpoint and response to clarify what is supported in a certain context.

## Using Standard Header definitions

Usually, you can the standard HTTP request and response header definition
provided by the guideline to simplify API by using well recognized patterns.
The best practice importing headers providing the highest readability is as
follows:

```yaml
path:
  '/resource'
    get:
      parameters:
        - $ref: '#/components/parameters/ETag

components:
  parameters|headers:
    ETag:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/ETag'

  responses:
    Default:
      headers:
        ETag:
          $ref: '#/components/(parameters|headers)/ETag'
```

**Note:** It is a question of taste whether headers for responses are defined in
`#/components/headers` or `#/components/parameters`. Unfortunately, headers
in the first section cannot be referenced from a `parameters`-list, while it is
possible to reference the second also from a `headers`-list.


## MAY use standard headers {#rule-133}

APIs may make use of HTTP headers defined by non-obsolete RFCs (see following
list of [standard HTTP headers](http://en.wikipedia.org/wiki/List_of_HTTP_header_fields)). The supported headers must be explicitly mentioned in the API
specification.


## SHOULD use kebab-case with uppercase separate words for HTTP headers {#rule-132}

This convention is followed by (most of) the standard headers e.g. as defined
in [RFC 2616](https://tools.ietf.org/html/rfc2616), [RFC 4229](https://tools.ietf.org/html/rfc4229), [RFC 9110](https://tools.ietf.org/html/rfc9110). Examples:

```http
If-Modified-Since
Accept-Encoding
Content-ID
Language
```

**Note:** The HTTP standard defines headers (now called HTTP fields) to be
case-insensitive (see [RFC 9110 Section 5.1](https://tools.ietf.org/html/rfc9110#section-5.1)). However,
for the sake of readability and consistency, you should follow the convention
when defining standard or proprietary headers. Exceptions are common
abbreviations like `ID`.


## MUST use `Content-*` headers correctly {#rule-178}

Content or entity headers are headers with a `Content-` prefix. They describe
the content of the body of the message and they can be used in both, HTTP
requests and responses. Commonly used content headers include but are not
limited to:

- `Content-Disposition` can indicate that the representation is supposed to be
  saved as a file, and the proposed file name.
- `Content-Encoding` indicates compression or encryption algorithms applied to
  the content.
- `Content-Length` indicates the length of the content (in bytes).
- `Content-Language` indicates that the body is meant for people literate in
  some human language(s).
- `Content-Location` indicates where the body can be found otherwise (#179
  for more details]).
- `Content-Range` is used in responses to range requests to indicate which part
  of the requested resource representation is delivered with the body.
- `Content-Type` indicates the media type of the body content.


## SHOULD use `Location` header instead of `Content-Location` header {#rule-180}

As a correct usage of the `Content-Location` response header (see #179)
with respect to caching and its method specific semantics is difficult, we
**discourage** the use of `Content-Location`. In most cases it is sufficient to
inform clients about the resource location in create or re-direct responses by
using the `Location` header while avoiding the `Content-Location` specific
ambiguities and complexities.

For more details see [RFC 9110 Section 10.2.2 Location](https://tools.ietf.org/html/rfc9110#section-10.2.2), [RFC 9110 Section 8.2 Content-Location](https://tools.ietf.org/html/rfc9110#section-8.7)


## MAY use `Content-Location` header {#rule-179}

The `Content-Location` response header is an *optional* header that can be used
in successful write operations (`PUT`, `POST`, or `PATCH`) and read operations
(`GET`, `HEAD`) to guide caching and signal a receiver the actual location of
the resource transmitted in the response body. This allows clients to identify
the resource and to update their local copy when receiving a response with this
header.

The `Content-Location` header can be used to support the following use cases:

- For reading operations `GET` and `HEAD`, a different location than the
  requested URL can be used to indicate that the returned resource is subject
  to content negotiations (#244), and that the value provides a more specific
  identifier of the resource.
- For writing operations `PUT` and `PATCH`, an identical location to the
  requested URL can be used to explicitly indicate that the returned resource
  is the current representation of the newly created or updated resource.
- For writing operations `POST` and `DELETE`, a content location can be used to
  indicate that the body contains a status report resource in response to the
  requested action, which is available at provided location.

**Note:** When using the `Content-Location` header, the `Content-Type` header
has to be set as well. For example:

```http
GET /products/123/images HTTP/1.1

HTTP/1.1 200 OK
Content-Type: image/png
Content-Location: /products/123/images?format=raw
```

See also #227.


## MAY consider to support `Prefer` header to handle processing preferences {#rule-181}

The `Prefer` header defined in [RFC 7240](https://tools.ietf.org/html/rfc7240) allows clients to request
processing behaviors from servers. It pre-defines a number of preferences and
is extensible, to allow others to be defined. Support for the `Prefer` header
is entirely optional and at the discretion of API designers, but as an existing
Internet Standard, is recommended over defining proprietary "X-" headers for
processing directives.

The `Prefer` header can be defined in the API specification as follows:

```yaml
components:
  parameters:
    Prefer:
        # Do not import this schema directly, since processing directives are usually
        # highly customized. Instead, copy the schema to your API and adjust it to
        # your needs.
        name: Prefer
        in: header
        required: false
        description: |-
          The **Prefer** header indicates that a particular server behavior is
          preferred by the client, but is not required for successful completion of
          the request (see [RFC 7240][rfc-7240]. The following behaviors are
          supported by this API:

          * **respond-async** is used to suggest the server to respond as fast as
            possible asynchronously using **202** (Accepted) instead of waiting for
            the result.
          * **return=<minimal|representation>** is used to suggest the server to
            return using **204** (No Content) without resource (minimal) or using
            **200** or **201** with resource (representation) in the response body on
            success.
          * **return=<total-count>** is used to suggest the server to return a total
            result count in a collection requests supporting pagination. Since this
            is a costly operation, it should be used with care, and the service may
            decide to ignore this request.
          * **wait=<delta-seconds>** is used to suggest a maximum time the server has
            time to process the request synchronously.
          * **handling=<strict|lenient>** is used to suggest the server to be strict
            and report error conditions or lenient, i.e. robust and try to continue,
            if possible.

          See [RFC 7240][rfc-7240] as well as [API Guideline Rule #181][api-181]
          for further details.

          [rfc-7240]: <https://tools.ietf.org/html/rfc7240>
          [api-181]: <https://opensource.zalando.com/restful-api-guidelines/#181>
        schema:
          type: array
          items:
            type: string
        style: simple
        explode: false
        example: [ "respond-async", "return=minimal", "wait=20", "handling=strict",
          "respond-async,return=minimal,wait=20,handling=strict" ]
```

**Note:** Please, copy only the behaviors into your `Prefer` header specification
that are supported by your API endpoint. If necessary, specify different
`Prefer` headers for each supported use case.

Supporting APIs may return the `Preference-Applied` header also defined in
[RFC 7240](https://tools.ietf.org/html/rfc7240) to indicate whether a preference has been applied.


## MAY consider to support `ETag` together with `If-Match`/`If-None-Match` header {#rule-182}

When creating or updating resources it may be necessary to expose conflicts
and to prevent the *lost update* or *initially created* problem. Following
[RFC 9110 Section 13 "Conditional Requests"](https://tools.ietf.org/html/rfc9110) this can be best
accomplished by supporting the `ETag` header together with the `If-Match` or
`If-None-Match` conditional header. The contents of an `ETag: <entity-tag>`
header is either (a) a hash of the response body, (b) a hash of the last
modified field of the entity, or (c) a version number or identifier of the
entity version.

To expose conflicts between concurrent update operations via `PUT`, `POST`, or
`PATCH`, the `If-Match: <entity-tag>` header can be used to force the server to
check whether the version of the updated entity is conforming to the requested
`<entity-tag>`. If no matching entity is found, the operation is supposed a to
respond with status code 412 - precondition failed.

Beside other use cases, `If-None-Match: *` can be used in a similar way to
expose conflicts in resource creation. If any matching entity is found, the
operation is supposed a to respond with status code 412 - precondition
failed.

The `ETag`, `If-Match`, and `If-None-Match` headers can be defined as follows
in the API specification (see also the default definition below):

```yaml
components:
  parameters|headers:
    ETag:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/ETag'
    If-Match:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/If-Match'
    If-None-Match:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/If-None-Match'
```

```yaml
components:
  headers:
    ETag:
        required: false
        description: |-
          The **ETag** header field in a response provides an opaque quoted string
          identifying the distinct delivered resource. The same selected resource
          depending on version and representation may be identified by multiple
          identifiers. The **ETag** value is guaranteed to change whenever the
          resource changes, and thereby enabling optimistic updates.

          An identifier consists of an opaque quoted string, possibly prefixed by
          a weakness indicator. See [RFC 9110 Section 8.8.3][rfc-9110-8.8.3] as
          well as [API Guideline Rule #182][api-182] for further details.

          [rfc-9110-8.8.3]: <https://tools.ietf.org/html/rfc9110#section-8.8.3>
          [api-182]: <https://opensource.zalando.com/restful-api-guidelines/#182>
        schema:
          type: array
          items:
            type: string
        style: simple
        explode: false
        example: [ W/"xy", "5", "5db68c06-1a68-11e9-8341-68f728c1ba70" ]
    If-Match:
        name: If-Match
        in: header
        required: false
        description: |-
          The **If-Match** header field is used to declare a list of identifiers that
          are required to match the current resource version identifier in at least
          one position as a pre-condition for executing the request on the server
          side. This behavior is used to validate and reject optimistic updates, by
          checking if the resource version a consumer has based his changes on is
          outdated on arrival of the change request to prevent lost updates.

          If the pre-condition fails the server will respond with status code **412**
          (Precondition Failed). See [RFC 9110 Section 13.1.1][rfc-9110-13.1.1] as
          well as [API Guideline Rule #182][api-182] for further details.

          [rfc-9110-13.1.1]: <https://tools.ietf.org/html/rfc9110#section-13.1.1>
          [api-182]: <https://opensource.zalando.com/restful-api-guidelines/#182>
        schema:
          type: array
          items:
            type: string
        style: simple
        explode: false
        example: [ "5db68c06-1a68-11e9-8341-68f728c1ba70", 'W/"xy", "5"' ]
    If-None-Match:
        name: If-None-Match
        in: header
        required: false
        description: |-
          The **If-None-Match header** field is used to declare a list of identifiers
          that are required to fail matching all the current resource version
          identifiers as a pre-condition for executing the request on the server
          side. This is especially used in conjunction with an **\*** (asterix) that
          is matching all possible resource identifiers to ensure the initial
          creation of a resource. Other use cases are possible but rare.

          If the pre-condition fails the server will respond with status code **412**
          (Precondition Failed). See [RFC 9110 Section 13.1.2][rfc-9110-13.1.2] as
          well as [API Guideline Rule #182][api-182] for further details.

          [rfc-9110-13.1.2]: <https://tools.ietf.org/html/rfc9110#section-13.1.2>
          [api-182]: <https://opensource.zalando.com/restful-api-guidelines/#182>
        schema:
          type: array
          items:
            type: string
        style: simple
        explode: false
        example: [ "5db68c06-1a68-11e9-8341-68f728c1ba70", 'W/"xy", "5"' ]
```

Please see the *Optimistic Locking* section for a detailed discussion as well as
the *Cache Support Patterns* section for additional use cases.


## MAY consider to support `Idempotency-Key` header {#rule-230}

When creating or updating resources it can be helpful or necessary to ensure a
strong *idempotent* behavior comprising same responses, to prevent duplicate
execution in case of retries after timeout and network outages. Generally, this
can be achieved by sending a client specific *unique request key* – that is not
part of the resource – via `Idempotency-Key` header.

The *unique request key* is stored temporarily, e.g. for 24 hours, together
with the response and the request hash (optionally) of the first request in a
key cache, regardless of whether it succeeded or failed. The service can now
look up the *unique request key* in the key cache and serve the response from
the key cache, instead of re-executing the request, to ensure *idempotent*
behavior. Optionally, it can check the request hash for consistency before
serving the response. If the key is not in the key store, the request is
executed as usual and the response is stored in the key cache.

This allows clients to safely retry requests after timeouts, network outages,
etc. while receive the same response multiple times. **Note:** The request retry
in this context requires to send the exact same request, i.e. updates of the
request that would change the result are off-limits. The request hash in the
key cache can protection against this misbehavior. The service is recommended
to reject such a request using status code 400.

**Important:** To grant a reliable *idempotent* execution semantic, the
resource and the key cache have to be updated with hard transaction semantics
– considering all potential pitfalls of failures, timeouts, and concurrent
requests in a distributed systems. This makes a correct implementation
exceeding the local context very hard.

The `Idempotency-Key` header must be defined as follows, but you are free to
choose your expiration time:

```yaml
components:
  parameters:
    Idempotency-Key:
        name: Idempotency-Key
        in: header
        required: false
        description: |-
          The **Idempotency-Key** is a free identifier created by the client to
          identify a request. It is used by the service to identify repeated request
          to ensure idempotent behavior by sending the same (or a similar) response
          without executing the request a second time.

          Clients should be careful as any subsequent requests with the same key may
          return the same response without further check. Thus, it is recommended to
          use a UUID version 4 (random) or any other random string with enough
          entropy to avoid collisions.

          Keys expire after 24 hours. Clients are responsible to stay within this
          limit, if they require idempotent behavior.

          See [API Guideline Rule #181][api-230] for further details.

          [api-230]: <https://opensource.zalando.com/restful-api-guidelines/#230>
        schema:
          type: string
          format: uuid
        example: [ "7da7a728-f910-11e6-942a-68f728c1ba70" ]
```

If you do not want to change the expiration time, you can also use the standard
definition provided by the guideline:

```yaml
components:
  parameters|headers:
    Idempotency-Key:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/Idempotency-Key'
```

**Hint:** The key cache is not intended as request log, and therefore should
have a limited lifetime, else it could easily exceed the data resource in
size.

**Note:** The `Idempotency-Key` header unlike other headers in this section
is not standardized in an RFC. Our only reference are the usage in the
[Stripe API](https://stripe.com/docs/api/idempotent_requests).
However, we do not want to change the header name and semantic, and
do not name it like the proprietary headers below.
The header addresses a generic REST concern and is different from the
Zalando landscape specific proprietary headers.


## SHOULD use only the specified proprietary Zalando headers {#rule-183}

As a general rule, proprietary HTTP headers should be avoided. In addition
from a conceptual point of view, the business semantics and intent of an
operation should always be expressed via the path and query parameters, the
method, and the content, but not via proprietary headers.

Headers are typically used to implement protocol processing aspects, such as
flow control, content negotiation, and authentication, and represent business
agnostic request modifiers that provide generic context information
([RFC 9110 Section 10 "Message Context"](https://tools.ietf.org/html/rfc9110#section-10)).

However, the exceptional usage of proprietary headers is still helpful when
domain-specific generic context information

1. needs to be passed *end-to-end* along the service call chain (even if not
   all called services use it as input for steering service behavior, e.g.
   `X-Sales-Channel` header), and/or
2. is provided by specific gateway components, for instance, our Fashion Shop
   API or Merchant API gateway.

Below, we explicitly define the list of proprietary headers usable for all
services for passing through generic context information of our fashion domain
(use case 1).

Per convention, non standardized, proprietary header names are prefixed with
`X-` and use the dash (`-`) as separator (dash-case). (Due to backward
compatibility, we do not follow the recommendation of Internet Engineering
Task Force in [RFC 6648](https://tools.ietf.org/html/rfc6648) to deprecate the usage of  `X-`headers.)
Remember that HTTP header field names are not case-sensitive:

| Header field name | Type | Description | Header field value example |
|---|---|---|---|
| `X-Flow-ID` | String | For more information see #233. | GKY7oDhpSiKY_gAAAABZ_A |
| `X-Tenant-ID` | String | Identifies the tenant initiated the request to the multi tenant Zalando Platform. The `X-Tenant-ID` must be set according to the Business Partner ID extracted from the OAuth token when a request from a Business Partner hits the Zalando Platform. | 9f8b3ca3-4be5-436c-a847-9cd55460c495 |
| `X-Sales-Channel` | String | Sales channels are owned by retailers and represent a specific consumer segment being addressed with a specific product assortment that is offered via CFA retailer catalogs to consumers (see [fashion platform glossary (internal link)](https://digital-experience.docs.zalando.net/glossary/glossary.html)). | 52b96501-0f8d-43e7-82aa-8a96fab134d7 |
| `X-Frontend-Type` | String | Consumer facing applications (CFAs) provide business experience to their customers via different frontend application types, for instance, mobile app or browser. Info should be passed-through as generic aspect -- there are diverse concerns, e.g. pushing mobiles with specific coupons, that make use of it. Current range is mobile-app, browser, facebook-app, chat-app, email. | mobile-app |
| `X-Device-Type` | String | There are also use cases for steering customer experience (incl. features and content) depending on device type. Via this header info should be passed-through as generic aspect. Current range is smartphone, tablet, desktop, other. | tablet |
| `X-Device-OS` | String | On top of device type above, we even want to differ between device platform, e.g. smartphone Android vs. iOS. Via this header info should be passed-through as generic aspect. Current range is iOS, Android, Windows, Linux, MacOS. | Android |
| `X-Mobile-Advertising-ID` | String | It is either the [IDFA](https://developer.apple.com/documentation/adsupport/asidentifiermanager) (Apple Identifier for mobile Advertising) for iOS, or the [GAID](https://support.google.com/googleplay/android-developer/answer/6048248) (Google mobile Advertising Identifier) for Android. It is a unique, customer-resettable identifier provided by mobile device’s operating system to facilitate personalized advertising, and usually passed by mobile apps via HTTP header when calling backend services. Called services should be ready to pass this parameter through when calling other services. It is not sent if the customer disables it in the settings for respective mobile platform. | b89fadce-1f42-46aa-9c83-b7bc49e76e1f |

**Exception:** The only exception to this guideline are the conventional
hop-by-hop `X-RateLimit-` headers which can be used as defined in #153.

As part of the guidelines we provide the default definition of all proprietary
headers, so you can simply reference them when defining the API endpoint. For
details see the *Using Standard Header definitions* section.

**Hint:** This guideline does not standardize proprietary headers for our
specific gateway components (2. use case above). This include, for instance,
non pass-through headers `X-Zalando-Customer`, `X-Zalando-Client-ID`,
`X-Zalando-Request-Host`, `X-Zalando-Request-URI` defined by Fashion Shop API
(RKeep), or `X-Consumer`, `X-Consumer-Signature`, `X-Consumer-Key-ID` defined
by Merchant API gateway, or `X-App-Version`, `X-Country-Code`, `X-Zalando-Auth`,
`X-Forwarded-For` defined by Transactions Checkout Platform. All these proprietary
headers are allow-listed in the API Linter (Zally) checking this rule.


## MUST propagate proprietary headers {#rule-184}

All Zalando's proprietary headers defined in #183 are end-to-end headers[^header-types] and must be propagated to the services down the call
chain. The header names and values must remain unchanged.

The values of custom headers can influence query results (e.g. `X-Device-Type` can
affect recommendation results by conveying device‑type information).

Sometimes the value of a proprietary header will be used as part of the entity
in a subsequent request. In such cases, the proprietary headers must still be
propagated as headers with the subsequent request, despite the duplication of
information.

[^header-types]: HTTP/1.1 standard ([RFC 9110 Section 7.6.1](https://tools.ietf.org/html/rfc9110#section-7.6.1)) defines two types of headers: *end-to-end* and *hop-by-hop* headers. End-to-end headers must be transmitted to the ultimate recipient of a request or response. Hop-by-hop headers, on the contrary, are meaningful for a single connection only.


## MUST support `X-Flow-ID` {#rule-233}

The `Flow-ID` is a generic parameter to be passed through service APIs and
events and written into log files and traces. A consequent usage of the
`Flow-ID` facilitates the tracking of call flows through our system and allows
the correlation of service activities initiated by a specific call. This is
extremely helpful for operational troubleshooting and log analysis. Main use
case of `Flow-ID` is to track service calls of our SaaS fashion commerce
platform and initiated internal processing flows (executed synchronously via
APIs or asynchronously via published events).


### Data Definition

The `Flow-ID` must be passed through:

- RESTful API requests via `X-Flow-ID` proprietary header (see #184)
- Published events via `flow_id` event field (see the *metadata* section)

The following formats are allowed:

- `UUID` ([RFC-4122](https://tools.ietf.org/html/rfc4122))
- `base64` ([RFC-4648](https://tools.ietf.org/html/rfc4648))
- `base64url` ([RFC-4648 Section 5](https://tools.ietf.org/html/rfc4648#section-5))
- Random unique string restricted to the character set `[a-zA-Z0-9/+_-=]`
  maximal of 128 characters.

**Note:** If a legacy subsystem can only process `Flow-IDs` with a specific
format or length, it must define this restriction in its API specification,
and be generous and remove invalid characters or cut the length to the
supported limit.

**Hint:** In case distributed tracing is supported by
[OpenTracing (internal link)](https://github.bus.zalan.do/SRE/opentracing) you should ensure that created
*spans* are tagged using `flow_id` — see
[How to Connect Log Output with OpenTracing Using Flow-IDs (internal link)](https://github.bus.zalan.do/SRE/opentracing/blob/master/wg-semantic-conventions/best-practices/flowid.md)  or
[Best practices (internal link)](https://github.bus.zalan.do/SRE/opentracing/blob/master/wg-semantic-conventions/best-practices.md).


### Service Guidance

- Services **must** support `Flow-ID` as generic input, i.e.
  - RESTful API endpoints **must** support `X-Flow-ID` header in requests
  - Event listeners **must** support the metadata `flow-id` from events.


**Note:** API-Clients **must** provide `Flow-ID` when calling a service or
producing events. If no `Flow-ID` is provided in a request or event, the
service must create a new `Flow-ID`.

- Services **must** propagate `Flow-ID`, i.e. use `Flow-ID` received
  with API calls or consumed events as...
  - input for all API called and events published during processing
  - data field written for logging and tracing

**Hint:** This rule also applies to application internal interfaces and events
not published via Nakadi (but e.g. via AWS SQS, Kinesis or service specific
DB solutions).
