# REST Basics - HTTP status codes


## MUST use official HTTP status codes {#rule-243}

You must only use official HTTP status codes consistently with their intended
semantics. Official HTTP status codes are defined via RFC standards and
registered in the
[IANA Status Code Registry](https://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml). Main RFC standards are [RFC 9110 - HTTP Semantics, section 15: Status Codes](https://tools.ietf.org/html/rfc9110#name-status-codes) and
[RFC 6585 - HTTP: Additional Status Codes](https://tools.ietf.org/html/rfc6585).
[Wikipedia: HTTP status codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)
provides an overview on the official error codes (which also lists some unofficial
status codes, e.g. defined by popular web servers like Nginx, that we do not suggest to use).


## MUST specify success and error responses {#rule-151}

APIs should define the functional, business view and abstract from
implementation aspects. Success and error responses are a vital part to
define how an API is used correctly.

Therefore, you must define **all** success and service specific error responses
in your API specification. Both are part of the interface definition and
provide important information for service clients to handle standard as well as
exceptional situations. Error code response descriptions should provide
information about the specific conditions that lead to the error, especially if
these conditions can be changed by how the endpoint is used by the clients.

API designers should also think about a **troubleshooting board** as part of
the associated online API documentation. It provides information and handling
guidance on application-specific errors and is referenced via links from the
API specification. This can reduce service support tasks and contribute to
service client and provider performance.

**Exception:** Standard client and server errors, e.g. `401` (unauthenticated),
`403` (unauthorized), `404` (not found), `500` (internal server error), or
`503` (service unavailable), where the semantic can be easily derived from the
end endpoint specification need no individual definition. Instead these can be
included by applying the `default` shown pattern below. However, error codes
that provide endpoint specific indications for clients on how to avoid calling
the endpoint in the wrong way, or be prepared to react on specific error
situation must be specified explicitly.

```yaml
responses:
  ...
  default:
    description: error occurred - see status code and problem object for more information.
    content:
      "application/problem+json":
        schema:
          $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/problem-1.0.1.yaml#/Problem'
```


## SHOULD only use most common HTTP status codes {#rule-150}

The most commonly used codes are best understood and listed below as subset of
the official HTTP status codes and consistent with their semantics in the RFCs.
We avoid less commonly used codes that easily create misconceptions due to
less familiar semantics and API specific interpretations.

**Important:** As long as your HTTP status code usage is well covered by the
semantic defined here, you should not describe it to avoid an overload with
common sense information and the risk of inconsistent definitions. Only if the
HTTP status code is not in the list below or its usage requires additional
information aside the well defined semantic, the API specification must provide
a clear description of the HTTP status code in the response.

### Legend

A more specific version of the *Conventions used in these guidelines* section.

#### use

A common, well understood status code that should be used where appropriate.
Does NOT mean that every operation must be able to return this code.

#### do not use

We do not see a good use-case for returning this status code from a RESTful
API. The status code might be applicable in other contexts, e.g. returned by
reverse proxies, for web pages, etc. Implicitly also means **do not document**,
as status codes that are not returned by the API should also not be documented.

#### document

If the status code can be returned by the API, it should be documented in the
API specification.

#### do not document

The status code has a well-understood standard meaning to it, so only document
it if there are operation specific details you want to add. See exception in
rule 151 ([#151](#rule-151)).


### Success codes

#### 200 OK

[RFC](https://tools.ietf.org/html/rfc9110#name-200-ok) · **use** · **document** · `<all>`

This is the most general success response. It should be used if the more
specific codes below are not applicable. In a robust resource creation via
`POST`, `PUT`, and `PATCH` in conjunction with a `201` this indicates that the
returned resource exists before.

#### 201 Created

[RFC](https://tools.ietf.org/html/rfc9110#name-201-created) · **use** · **document** · `POST` · `PUT`

Returned on successful resource creation.
`201` is returned with or without response payload (unlike `200` / `204`).
We recommend additionally to return the created resource URL via the `Location`
response header (see [#133](http-headers.md#rule-133)).

#### 202 Accepted

[RFC](https://tools.ietf.org/html/rfc9110#name-202-accepted) · **use** · **document** · `POST` · `PUT` · `PATCH` · `DELETE`

The request was successful and will be processed asynchronously.
Only applicable to methods which change something, with the exception of
`GET` methods indicating that a resources is still created asynchronously
as described in [#253](http-requests.md#rule-253).

#### 204 No content

[RFC](https://tools.ietf.org/html/rfc9110#name-204-no-content) · **use** · **document** · `POST` · `PUT` · `PATCH` · `DELETE`

Returned instead of `200`, if no response payload is returned. Normally only
applicable to methods which change something.

#### 205 Reset Content

[RFC](https://tools.ietf.org/html/rfc9110#name-205-reset-content) · **do not use** · `<all>`

This is meant for interactive use cases, e.g. to clear a form after submitting
it. There is no reason to use it in a REST API.

#### 206 Partial Content

[RFC](https://tools.ietf.org/html/rfc9110#name-206-partial-content) · **do not use** · `<all>`

For responses to range requests, when only a part of the resource indicated by
the byte range is returned. This is not for pagination, where a normal `200`
should be used. This might be useful in rare cases (like media streaming or
downloading large files), but most APIs don't need this.

#### 207 Multi-Status

[RFC](https://tools.ietf.org/html/rfc4918#section-11.1) · **use** · **document** · `POST` (`DELETE`)

The response body contains status information for multiple different parts of a
batch/bulk request (see [#152](#rule-152) for details). Normally used for `POST`, in some
cases also for `DELETE`.


### Redirection codes

#### 301 Moved Permanently

[RFC](https://tools.ietf.org/html/rfc9110#name-301-moved-permanently) · **do not use** · `<all>`

This and all future requests should be directed to the given URI. See also
[#251](#rule-251).

#### 302 Found

[RFC](https://tools.ietf.org/html/rfc9110#name-302-found) · **do not use** · `<all>`

This is a temporary redirect where some clients **MAY** change the request method
from `POST` to `GET`. Mainly used for dismissing and redirecting form
submissions in browsers. See also [#251](#rule-251).

#### 303 See Other

[RFC](https://tools.ietf.org/html/rfc9110#name-303-see-other) · **do not use** · `POST` · `PUT` · `PATCH` · `DELETE`

The response to the request can be found under another URI using a `GET`
method. A disambiguated version of `302` for the case where the client **MUST**
change the method to `GET`. See also [#251](#rule-251).

#### 304 Not Modified

[RFC](https://tools.ietf.org/html/rfc9110#name-304-not-modified) · **document** · `GET` · `HEAD`

Indicates that a conditional `GET` or `HEAD` request would have resulted in
`200` response, if it were not for the fact that the condition evaluated to
false, i.e. resource has not been modified since the date or version passed via
request headers `If-Modified-Since` or `If-None-Match`. For
`PUT`/`PATCH`/`DELETE` requests, use `412` instead.

#### 307 Temporary Redirect

[RFC](https://tools.ietf.org/html/rfc9110#name-307-temporary-redirect) · **do not use** · `<all>`

The response to the request can be found under another URI. A disambiguated
version of `302` where the client **MUST** keep the same method as the original
request. See also [#251](#rule-251).

#### 308 Permanent Redirect

[RFC](https://tools.ietf.org/html/rfc9110#name-308-permanent-redirect) · **do not use** · `<all>`

Similar to `307`, but the client should persist the new URI. Applicable more to
browsers. For APIs, the URI should be explicitly fixed at the source instead of
being implicitly kept in some state based on a previous redirect. See also
[#251](#rule-251).


### Client side error codes

#### 400 Bad Request

[RFC](https://tools.ietf.org/html/rfc9110#name-400-bad-request) · **use** · **document** · `<all>`

Unspecific client error indicating that the server cannot process the request
due to something that is perceived to be a client error (e.g. malformed request
syntax, invalid request). Should also be delivered in case of input payload
fails business logic / semantic validation (instead of using `422`).

#### 401 Unauthorized

[RFC](https://tools.ietf.org/html/rfc9110#name-401-unauthorized) · **use** · **do not document** · `<all>`

Actually **Unauthenticated**. The credentials are missing or not valid for the
target resource. For an API, this usually means that the provided token or
cookie is not valid. As this can happen for almost every endpoint, APIs should
normally not document this.

#### 403 Forbidden

[RFC](https://tools.ietf.org/html/rfc9110#name-403-forbidden) · **document** · `<all>`

The user is not authorized to use this resource. For an API, this can mean that
the request's token was valid, but was missing a scope for this endpoint. Or
that some object-specific authorization failed. The first case (missing scopes)
does not need to be documented, but the second (object-specific authorization)
should be documented with context explaining what conditions cause it.

#### 404 Not found

[RFC](https://tools.ietf.org/html/rfc9110#name-404-not-found) · **use** · **do not document** · `<all>`

The target resource was not found. This will be returned by most paths on most
APIs (with out being documented), and for endpoints with parameters when those
parameters cannot be map to an existing entity. For a `PUT` endpoint which only
supports updating existing resources, this might be returned if the resource
does not exist. Apart from these special cases, this does not need to be
documented.

#### 405 Method Not Allowed

[RFC](https://tools.ietf.org/html/rfc9110#name-405-method-not-allowed) · **document** · `<all>`

The request method is not supported for this resource. In theory, this can be
returned for all resources for all the methods except the ones documented.
Using this response code for an existing endpoint (usually with path
parameters) only makes sense if it depends on some internal resource state
whether a specific method is allowed, e.g. an order can only be canceled via
`DELETE` until the shipment leaves the warehouse. **Do not use it unless you
have such a special use case, but then make sure to document it, making it
clear why a resource might not support a method.**

#### 406 Not Acceptable

[RFC](https://tools.ietf.org/html/rfc9110#name-406-not-acceptable) · **use** · **do not document** · `<all>`

Resource only supports generating content with content-types that are not
listed in the `Accept` header sent in the request.

#### 408 Request timeout

[RFC](https://tools.ietf.org/html/rfc9110#name-408-request-timeout) · **do not use** · `<all>`

The server times out waiting for the request to arrive. For APIs, this should
not be used.

#### 409 Conflict

[RFC](https://tools.ietf.org/html/rfc9110#name-409-conflict) · **document** · `POST` · `PUT` · `PATCH` · `DELETE`

The request cannot be completed due to conflict with the current state of the
target resource. For example, you may get a `409` response when updating a
resource that is older than the existing one on the server, resulting in a
version control conflict. If this is used, it **MUST** be documented. For
successful robust creation of resources (`PUT` or `POST`) you should always
return `200` or `204` and not `409`, even if the resource exists already. If
any `If-*` headers cause a conflict, you should use `412` and not `409`. Only
applicable to methods which change something.

#### 410 Gone

[RFC](https://tools.ietf.org/html/rfc9110#name-410-gone) · **do not document** · `<all>`

The resource does not exist any longer. It did exist in the past, and will
most likely not exist in the future. This can be used, e.g. when accessing a
resource that has intentionally been deleted. This normally does not need to be
documented, unless there is a specific need to distinguish this case from the
normal `404`.

#### 411 Length Required

[RFC](https://tools.ietf.org/html/rfc9110#name-411-length-required) · **document** · `POST` · `PUT` · `PATCH`

The server requires a `Content-Length` header for this request. This is
normally only relevant for large media uploads. The corresponding header
parameter should be marked as required. If used, it **MUST** to be documented
(and explained).

#### 412 Precondition Failed

[RFC](https://tools.ietf.org/html/rfc9110#name-412-precondition-failed) · **use** · **do not document** · `PUT` · `PATCH` · `DELETE`

Returned for conditional requests, e.g. `If-Match` if the condition failed.
Used for optimistic locking. Normally only applicable to methods that change
something. For `HEAD`/`GET` requests, use `304` instead.

#### 415 Unsupported Media Type

[RFC](https://tools.ietf.org/html/rfc9110#name-415-unsupported-media-type) · **use** · **do not document** · `POST` · `PUT` · `PATCH`

The client did not provide a supported content-type for the request body. Only
applicable to methods with a request body.

#### 417 Expectation Failed

[RFC](https://tools.ietf.org/html/rfc9110#name-417-expectation-failed) · **do not use** · `<all>`

Returned when the client used an `Expect` header which the server does not
support. The only defined value for the `Expect` header is very technical and
does not belong in an API.

#### 418 I'm a teapot 🫖 (Unused)

[RFC](https://tools.ietf.org/html/rfc9110#name-418-unused) · **do not use** · `<all>`

Only use if you are implementing an API for a teapot that does not support
brewing coffee. Response defined for April's Fools in
[RFC 2324](https://www.rfc-editor.org/rfc/rfc2324.html).

#### 422 Unprocessable Content

[RFC](https://tools.ietf.org/html/rfc9110#name-422-unprocessable-content) · **do not use** · `<all>`

The server understands the content type, but is unable to process the content.
We do not recommend this code to be used as `400` already covers most use-cases
and there does not seem to be a clear benefit to differentiating between them.

#### 423 Locked

[RFC](https://tools.ietf.org/html/rfc4918#section-11.3) · **document** · `PUT` · `PATCH` · `DELETE`

Pessimistic locking, e.g. processing states. May be used to indicate an
existing resource lock, however, we recommend using optimistic locking instead.
If used, it must be documented to indicate pessimistic locking.

#### 424 Failed Dependency

[RFC](https://tools.ietf.org/html/rfc4918#section-11.4) · **do not use** · `<all>`

The request failed due to failure of a previous request. This is not applicable
to restful APIs.

#### 428 Precondition Required

[RFC](https://tools.ietf.org/html/rfc6585#section-7.1) · **use** · **do not document** · `<all>`

Server requires the request to be conditional, e.g. to make sure that the "lost
update problem" is avoided (see [#182](http-headers.md#rule-182)). Instead of documenting this response
status, the required headers should be documented (and marked as required).

#### 429 Too many requests

[RFC](https://tools.ietf.org/html/rfc6585#section-7.2) · **use** · **do not document** · `<all>`

The client is not abiding by the rate limits in place and has sent too many
requests (see [#153](#rule-153)).

#### 431 Request Header Fields Too Large

[RFC](https://tools.ietf.org/html/rfc6585#section-7.3) · **do not document** · `<all>`

The server is not able to process the request because the request headers are
too large. Usually used by gateways and proxies with memory limits.


### Server side error codes

#### 500 Internal Server Error

[RFC](https://tools.ietf.org/html/rfc9110#name-500-internal-server-error) · **use** · **do not document** · `<all>`

A generic error indication for an unexpected server execution problem. Clients
should be careful with retrying on this response, since the nature of the
problem is unknown and must be expected to continue.

#### 501 Not Implemented

[RFC](https://tools.ietf.org/html/rfc9110#name-501-not-implemented) · **document** · `<all>`

Server cannot fulfill the request, since the endpoint is not implemented yet.
Usually this implies future availability, but retrying now is not recommended.
May be documented on endpoints that are planned to be implemented in the
future to indicate that they are still not available.

#### 502 Bad Gateway

[RFC](https://tools.ietf.org/html/rfc9110#name-502-bad-gateway) · **use** · **do not document** · `<all>`

The server, while acting as a gateway or proxy, received an invalid response
from an inbound server attempting to fulfill the request. May be used by a
server to indicate that an inbound service is creating an unexpected result
instead of `500`. Clients should be careful with retrying on this response,
since the nature of the problem is unknown and must be expected to continue.

#### 503 Service Unavailable

[RFC](https://tools.ietf.org/html/rfc9110#name-503-service-unavailable) · **use** · **do not document** · `<all>`

Service is (temporarily) not available, e.g. if a required component or inbound
service is not available. Client are encouraged to retry requests following an
exponential back off pattern. If possible, the service should indicate how long
the client should wait by setting the `Retry-After` header.

#### 504 Gateway Timeout

[RFC](https://tools.ietf.org/html/rfc9110#name-504-gateway-timeout) · **use** · **do not document** · `<all>`

The server, while acting as a gateway or proxy, did not receive a timely
response. May be used by servers to indicate that an inbound service cannot
process the request fast enough. Client may retry the request immediately
exactly once, to check whether warming up the service solved the problem.

#### 505 HTTP Version Not Supported

[RFC](https://tools.ietf.org/html/rfc9110#name-505-http-version-not-suppor) · **do not use** · `<all>`

The server does not support the HTTP protocol version used in the request.
Technical response code that serves no use case in RESTful APIs.

#### 507 Insufficient Storage

[RFC](https://tools.ietf.org/html/rfc4918#section-11.5) · **do not document** · `POST` · `PUT` · `PATCH`

The server is unable to store the resource as needed to complete the request.
May be used to indicate that the server is out of disk space.

#### 511 Network Authentication Required

[RFC](https://tools.ietf.org/html/rfc6585#section-7.4) · **do not use** · `<all>`

The client needs to authenticate to gain network access. Technical response
code that serves no use case in RESTful APIs.


## MUST use most specific HTTP status codes {#rule-220}

You must use the most specific HTTP status code when returning information
about your request processing status or error situations.


## MUST use code 207 for batch or bulk requests {#rule-152}

Some APIs are required to provide either *batch* or *bulk* requests using
`POST` for performance reasons, i.e. for communication and processing
efficiency. In this case services may be in need to signal multiple response
codes for each part of a batch or bulk request. As HTTP does not provide
proper guidance for handling batch/bulk requests and responses, we herewith
define the following approach:

- A batch or bulk request **always** responds with HTTP status code `207`
  unless a non-item-specific failure occurs.

- A batch or bulk request **may** return `4xx`/`5xx` status codes, if the
  failure is non-item-specific and cannot be restricted to individual items of
  the batch or bulk request, e.g. in case of overload situations or general
  service failures.

- A batch or bulk response with status code `207` **always** returns as payload
  a multi-status response containing item specific status and/or monitoring
  information for each part of the batch or bulk request.

**Note:** These rules apply *even in the case* that processing of all
individual parts *fail* or each part is executed *asynchronously*!

The rules are intended to allow clients to act on batch and bulk responses in
a consistent way by inspecting the individual results. We explicitly reject
the option to apply `200` for a completely successful batch as a short cut
without inspecting the result, as we
want to avoid risks and expect clients to handle partial
batch failures anyway.

The bulk or batch response may look as follows:

```yaml
BatchOrBulkResponse:
  description: batch response object.
  type: object
  properties:
    items:
      type: array
      items:
        type: object
        properties:
          id:
            description: Identifier of batch or bulk request item.
            type: string
          status:
            description: >
              Response status value. A number or extensible enum describing
              the execution status of the batch or bulk request items.
            type: string
            x-extensible-enum: [...]
          description:
            description: >
              Human readable status description and containing additional
              context information about failures etc.
            type: string
        required: [id, status]
```

**Note**: while a *batch* defines a collection of requests triggering
independent processes, a *bulk* defines a collection of independent
resources created or updated together in one request. With respect to
response processing this distinction normally does not matter.


## MUST use code 429 with headers for rate limits {#rule-153}

APIs that wish to manage the request rate of clients must use the `429` (Too
Many Requests) response code, if the client exceeded the request rate (see
[RFC 6585](https://tools.ietf.org/html/rfc6585)). Such responses must also contain header information
providing further details to the client. There are two approaches a service
can take for header information:

- Return a `Retry-After` header indicating how long the client ought to wait
  before making a follow-up request. The Retry-After header can contain a HTTP
  date value to retry after or the number of seconds to delay. Either is
  acceptable but APIs should prefer to use a delay in seconds.
- Return a trio of `X-RateLimit` headers. These headers (described below) allow
  a server to express a service level in the form of a number of allowing
  requests within a given window of time and when the window is reset.

The `X-RateLimit` headers are:

- `X-RateLimit-Limit`: The maximum number of requests that the client is
  allowed to make in this window.
- `X-RateLimit-Remaining`: The number of requests allowed in the current
  window.
- `X-RateLimit-Reset`: The relative time in seconds when the rate limit window
  will be reset. **Beware** that this is different to Github and Twitter's
  usage of a header with the same name which is using UTC epoch seconds
  instead.

The reason to allow both approaches is that APIs can have different
needs. Retry-After is often sufficient for general load handling and
request throttling scenarios and notably, does not strictly require the
concept of a calling entity such as a tenant or named account. In turn
this allows resource owners to minimize the amount of state they have to
carry with respect to client requests. The 'X-RateLimit' headers are
suitable for scenarios where clients are associated with pre-existing
account or tenancy structures. 'X-RateLimit' headers are generally
returned on every request and not just on a 429, which implies the
service implementing the API is carrying sufficient state to track the
number of requests made within a given window for each named entity.


## MUST support problem JSON {#rule-176}

[RFC 9457](https://tools.ietf.org/html/rfc9457) defines a Problem JSON object using the media type
`application/problem+json` to provide an extensible human and machine readable
failure information beyond the HTTP response status code to transports the
failure kind (`type` / `title`) and the failure cause and location (`instant` /
`detail`). To make best use of this additional failure information, every
endpoints must be capable of returning a Problem JSON on client usage errors
(`4xx` status codes) as well as server side processing errors (`5xx` status
codes).

**Note:** Clients must be robust and **not rely** on a Problem JSON object
being returned, since (a) failure responses may be created by infrastructure
components not aware of this guideline or (b) service may be unable to comply
with this guideline in certain error situations.

**Hint:** The media type `application/problem+json` is often not implemented as
a subset of `application/json` by libraries and services! Thus clients need to
include `application/problem+json` in the `Accept`-Header to trigger delivery
of the extended failure information.

The OpenAPI schema definition of the Problem JSON object can be found
[on GitHub](https://opensource.zalando.com/restful-api-guidelines/models/problem-1.0.1.yaml). You can reference it by using:

```yaml
responses:
  503:
    description: Service Unavailable
    content:
      "application/problem+json":
        schema:
          $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/problem-1.0.1.yaml#/Problem'
```

You may define custom problem types as extensions of the Problem JSON object
if your API needs to return specific, additional, and more detailed error
information.

**Note:** Problem `type` and `instance` identifiers in our APIs are not meant
to be resolved. [RFC 9457](https://tools.ietf.org/html/rfc9457) encourages that problem types are URI
references that point to human-readable documentation, **but** we deliberately
decided against that, as all important parts of the API must be documented
using OpenAPI ([#101](general-guidelines.md#rule-101)) anyway. In addition, URLs tend to be fragile and not
very stable over longer periods because of organizational and documentation
changes and descriptions might easily get out of sync.

In order to stay compatible with [RFC 9457](https://tools.ietf.org/html/rfc9457) we proposed to use
[relative URI references](https://tools.ietf.org/html/rfc3986#section-4.1)
usually defined by `absolute-path [ '?' query ] [ '#' fragment ]` as simplified
identifiers in `type` and `instance` fields:

- `/problems/out-of-stock`
- `/problems/insufficient-funds`
- `/problems/user-deactivated`
- `/problems/connection-error#read-timeout`

**Hint:** The use of [absolute URIs](https://tools.ietf.org/html/rfc3986#section-4.3) is not forbidden but strongly discouraged. If you use absolute URIs,
please reference
[problem-1.0.1.yaml#/Problem](https://opensource.zalando.com/restful-api-guidelines/models/problem-1.0.1.yaml#/Problem)
instead.


## MUST not expose stack traces {#rule-177}

Stack traces contain implementation details that are not part of an API,
and on which clients should never rely. Moreover, stack traces can leak
sensitive information that partners and third parties are not allowed to
receive and may disclose insights about vulnerabilities to attackers.

## SHOULD not use redirection codes {#rule-251}

We generally do not recommend using redirection codes for most API cases (except
for `304`, which is not really a redirection code). Usually you would use the
redirection to migrate clients to a new service location. However, this is
better accomplished by one of the following.

1. Changing the clients to use the new location in the first place, avoiding
   the need for redirection.
2. Redirecting the traffic behind the API layer (e.g. in the reverse proxy or
   the app itself) without the client having to be involved.
3. Deprecating the endpoint and removing it as described in the *Deprecation* section.

For idempotent `POST` cases, where you want to inform the client that a resource
already exists at a certain location, you should instead use `200` with the
`Location` header set. This is along the same lines as the creation case where
`201` is used instead. See also [#229](http-requests.md#rule-229).

For non-idempotent `POST` cases, where you want to inform the client that the
resource has already been created and cannot be created again (e.g. payment),
you should return `409` instead of redirecting to make the error case
more explicit.
