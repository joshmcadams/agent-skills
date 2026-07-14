# REST Basics - HTTP requests


## MUST use HTTP methods correctly {#rule-148}

Be compliant with the standardized HTTP semantics (see [RFC-9110 "HTTP semantics"](https://tools.ietf.org/html/rfc9110)) as summarized here.


### GET

`GET` requests are used to **read** either a single or a collection resource.

- `GET` requests for individual resources will usually generate a `404` (if the
  resource does not exist).
- `GET` requests for collection resources may return either `200` (if the
  collection is empty) or `404` (if the collection is missing).
- `GET` requests must NOT have a request body payload (see `GET with body`).

**Note:** `GET` requests on collection resources should provide sufficient
filter (#137) and pagination (see the *Pagination* section) mechanisms.


### GET with body payload

APIs sometimes face the problem, that they have to provide extensive structured
request information with `GET`, that may conflict with the size limits of
clients, load-balancers, and servers. As we require APIs to be standard conform
(request body payload in `GET` must be ignored on server side), API designers
have to check the following two options:

1. `GET` with URL encoded query parameters: when it is possible to encode the
   request information in query parameters, respecting the usual size limits of
   clients, gateways, and servers, this should be the first choice. The request
   information can either be provided via multiple query parameters or by a
   single structured URL encoded string.
2. `POST` with body payload content: when a `GET` with URL encoded query
   parameters is not possible, a `POST` request with body payload must be used,
   and explicitly documented with a hint like in the following example:

```yaml
paths:
  /products:
    post:
      description: >
        [GET with body payload](https://opensource.zalando.com/restful-api-guidelines/#get-with-body)
        - no resources created: Returns all products matching the query passed
        as request input payload.
      requestBody:
        required: true
        content:
          ...
```

**Note:** It is no option to encode the lengthy structured request information
using header parameters. From a conceptual point of view, the semantic of an
operation should always be expressed by the resource names, as well as the
involved path and query parameters. In other words by everything that goes into
the URL. Request headers are reserved for general context information (see
#183). In addition, size limits on query parameters and headers are not
reliable and depend on clients, gateways, server, and actual settings. Thus,
switching to headers does not solve the original problem.

**Hint:** As `GET with body` is used to transport extensive query parameters,
the `cursor` cannot any longer be used to encode the query filters in case of
cursor-based pagination (#160). As a consequence, it is best practice to
transport the query filters in the body payload, while keeping the
pagination parameters (`cursor`, `limit`) in query parameters.
This allows using pagination links (#161) containing the `cursor` that
is only encoding the page position and direction.
To protect the pagination sequence, the `cursor` may also contain a hash
over all applied query filters, which is then validated against the request
body. (See also #161.)


### PUT

`PUT` requests are used to **update** (and sometimes to create) **entire**
resources – single or collection resources. The semantic is best described
as *"please put the enclosed representation at the resource mentioned by
the URL, replacing any existing resource."*.

- `PUT` requests are usually applied to single resources, and not to collection
  resources, as this would imply replacing the entire collection.
- `PUT` requests are usually robust against non-existence of resources by
  implicitly creating the resource before updating.
- On successful `PUT` requests, the server will **replace the entire resource**
  addressed by the URL with the representation passed in the payload.
  Subsequent reads will deliver the same payload, plus possibly
  server-generated fields like `modified_at`.
- Successful `PUT` requests return `200` or `204` (if the resource was updated
  -- with or without returning the resource), `201` (if the resource was newly
  created), and `202` (if the request was accepted for asynchronous
  processing).

The updated/created resource may be returned as response payload. We recommend,
to not use it per default, but if the resource is enriched with
server-generated fields like `version_number`. You may also support client side
steering (see #181).

**Important:** It is good practice to keep the resource identifier management
under control of the service provider and not the client, and, hence, to prefer
`POST` for creation of (at least top-level) resources, and focus `PUT` on its
usage for updates. However, in situations where the identifier and all resource
attributes are under control of the client as input for the resource creation
you should use `PUT` and pass the resource identifier as URL path parameter.
Putting the same resource twice is required to be *idempotent* and to result
in the same single resource instance (see #149) without data duplication in
case of repetition.

**Hint:** To prevent unnoticed concurrent updates and duplicate creations when
using `PUT`, you #182 to allow the server to react on stricter demands that
expose conflicts and prevent lost updates. See also the *Optimistic Locking* section for
details and options.


### POST

`POST` requests are idiomatically used to **create** single resources on a
collection resource endpoint, but other semantics on single resources endpoint
are equally possible. The semantic for collection endpoints is best described as
*"please add the enclosed representation to the collection resource identified
by the URL"*. The semantic for single resource endpoints is best described as
*"please execute the given well specified request on the resource identified by
the URL"*.

- Successful `POST` requests on a collection endpoint will create one or more
  resource instances.
- For single resources, you **SHOULD** return `201` and the new resource object
  (including the resource identifier) in the response payload.
  - The client **SHOULD** be able to use this resource identifier to `GET` the
    resource if supported by the API (see #143).
  - The URL to `GET` the resource **SHOULD** be provided in the `Location` header.
  - If the response does not contain the created resource or a resource monitor,
    the status code **SHOULD** be `201` and the URL to `GET` the resource **MUST** be
    provided in the `Location` response header.
  - If the `POST` is *idempotent*, the status code **SHOULD** be `201` if the
    resource was created and `200` or `204` if the resource was updated.

> **Warning:** The resource identifier **MUST NOT** be passed as request body data by the client, but created and maintained by the service.

- For multiple resources you **SHOULD** return `201` in the response, as long as
  they are created atomically, meaning that either all the resources are created
  or none of them are.
  - You #152 if the request can partially fail, that is some of the resources
    can fail to be created.


> **Note:** Posting the same resource twice is **not** required to be *idempotent* (check #149) and may result in multiple resources. However, you #229 to prevent this.

- You #253, where the resource creation would not finish by the time the
  request is delivered, so the response status code **SHOULD** be `202`.


Apart from resource creation, `POST` should be also used for scenarios that
cannot be covered by the other methods sufficiently. However, in such cases
make sure to document the fact that `POST` is used as a workaround (see e.g.
`GET with body`).


### PATCH

`PATCH` method extends HTTP via [RFC-5789](https://tools.ietf.org/html/rfc5789) standard to update parts
of the resource objects where e.g. in contrast to `PUT` only a specific subset
of resource fields should be changed. The set of changes is represented in a
format called a *patch document* passed as payload and identified by a specific
media type. The semantic is best described as *"please change the resource
identified by the URL according to my patch document"*. The syntax and
semantics of the patch document is not defined in [RFC-5789](https://tools.ietf.org/html/rfc5789) and must
be described in the API specification by using specific media types.

- `PATCH` requests are usually applied to single resources as patching entire
  collection is challenging.
- `PATCH` requests are usually not robust against non-existence of resource
  instances.
- On successful `PATCH` requests, the server will update parts of the resource
  addressed by the URL as defined by the change request in the payload.
- Successful `PATCH` requests return `200` or `204` (if the resource was
  updated -- with or without returning the resource), and `202` (if the request
  was accepted for asynchronous processing).

**Note:** since implementing `PATCH` correctly is a bit tricky, we strongly
suggest to choose one and only one of the following patterns per endpoint
(unless forced by a backwards compatible change (#106)). In preference order:

1. Use `PATCH` with [JSON Merge Patch](https://tools.ietf.org/html/rfc7396) standard, a
   specialized media type `application/merge-patch+json` for partial
   resource representation to update parts of resource objects.
2. Use `PATCH` with [JSON Patch](https://tools.ietf.org/html/rfc6902) standard, a specialized media type
   `application/json-patch+json` that includes instructions on how to change
   the resource.
3. Use `POST` (with a proper description of what is happening) instead of
   `PATCH`, if the request does not modify the resource in a way defined by
   the semantics of the standard media types above.

**Note:** In earlier versions we proposed to avoid `PATCH` and use `PUT` with
complete objects to update a (sub)-resource as long as feasible, however, due
to the complexity this imposes on clients to prepare for compatible
extensions (#108), we reconsidered this approach in favor of solutions that
simplify clients while having a lower failure risk.

In practice [JSON Merge Patch](https://tools.ietf.org/html/rfc7396) quickly turns out to be too limited,
especially when trying to update single objects in large collections (as part
of the resource). In this case [JSON Patch](https://tools.ietf.org/html/rfc6902) is more powerful while
still showing readable patch requests (see also
[JSON patch vs. merge](http://erosb.github.io/post/json-patch-vs-merge-patch)).
JSON Patch supports changing of array elements identified via its index, but
not via (key) fields of the elements as typically needed for collections.

**Note:** Patching the same resource twice is **not** required to be *idempotent*
(check #149) and may result in a changing result. However, you #229 to
prevent this.

**Hint:** To prevent unnoticed concurrent updates when using `PATCH` you #182
to allow the server to react on stricter demands that expose conflicts and
prevent lost updates. See the *Optimistic Locking* section and #229 for details and
options.


### DELETE

`DELETE` requests are used to **delete** resources. The semantic is best
described as *"please delete the resource identified by the URL"*.

- `DELETE` requests are usually applied to single resources, not on
  collection resources, as this would imply deleting the entire collection.
- `DELETE` request can be applied to multiple resources at once using query
  parameters on the collection resource (see the *DELETE with query parameters* section).
- Successful `DELETE` requests return `200` or `204` (if the resource was
  deleted -- with or without returning the resource), or `202` (if the request
  was accepted for asynchronous processing).
- Failed `DELETE` requests will usually generate `404` (if the resource cannot
  be found) or `410` (if the resource was already traceably deleted before).

**Important:** After deleting a resource with `DELETE`, a `GET` request on the
resource is expected to either return `404` (not found) or `410` (gone)
depending on how the resource is represented after deletion. Under no
circumstances the resource must be accessible after this operation on its
endpoint.


### DELETE with query parameters

`DELETE` request can have query parameters. Query parameters should be used as
filter parameters on a resource and not for passing context information to
control the operation behavior.

```http
DELETE /resources?param1=value1&param2=value2...&paramN=valueN
```

**Note:** When providing `DELETE` with query parameters, API designers must
carefully document the behavior in case of (partial) failures to manage client
expectations properly.

The response status code of `DELETE` with query parameters requests should be
similar to usual `DELETE` requests. In addition, it may return the status code
`207` using a payload describing the operation results (see #152 for
details).


### DELETE with body payload

In rare cases `DELETE` may require additional information, that cannot be
classified as filter parameters and thus should be transported via request body
payload, to perform the operation. Since [RFC-9110 Section 9.3.5](https://tools.ietf.org/html/rfc9110#section-9.3.5) states, that `DELETE` has an undefined semantic for payloads, we
recommend to utilize `POST`. In this case the POST endpoint must be documented
with the hint `DELETE with body` analog to how it is defined for
`GET with body`. The response status code of `DELETE with body` requests should
be similar to usual `DELETE` requests.


### HEAD

`HEAD` requests are used to **retrieve** the header information of single
resources and resource collections.

- `HEAD` has exactly the same semantics as `GET`, but returns headers only, no
  body.

**Hint:** `HEAD` is particular useful to efficiently lookup whether large
resources or collection resources have been updated in conjunction with the
`ETag`-header.


### OPTIONS

`OPTIONS` requests are used to **inspect** the available operations (HTTP
methods) of a given endpoint.

- `OPTIONS` responses usually either return a comma separated list of methods
  in the `Allow` header or as a structured list of link templates.

**Note:** `OPTIONS` is rarely implemented, though it could be used to
self-describe the full functionality of a resource.


## MUST fulfill common method properties {#rule-149}

Request methods in RESTful services can be...

- [safe](https://tools.ietf.org/html/rfc9110#section-9.2.1) -- the operation semantic is defined to be read-only,
  meaning it must not have *intended side effects*, i.e. changes, to the server
  state.
- [idempotent](https://tools.ietf.org/html/rfc9110#section-9.2.2) -- the operation has the same
  *intended effect* on the server state, independently whether it is executed
  once or multiple times. **Note:** this does not require that the operation is
  returning the same response or status code.
- [cacheable](https://tools.ietf.org/html/rfc9110#section-9.2.3) -- to indicate that responses are
  allowed to be stored for future reuse. In general, requests to safe methods
  are cacheable, if it does not require a current or authoritative response
  from the server.

**Note:** The above definitions, of *intended (side) effect* allows the server
to provide additional state changing behavior as logging, accounting, pre-
fetching, etc. However, these actual effects and state changes, must not be
intended by the operation so that it can be held accountable.

Method implementations must fulfill the following basic properties according
to [RFC 9110 Section 9.2](https://tools.ietf.org/html/rfc9110#section-9.2):

| Method | Safe | Idempotent | Cacheable |
| --- | --- | --- | --- |
| `GET` | Yes | Yes | Yes |
| `HEAD` | Yes | Yes | Yes |
| `POST` | No | ⚠️ No, but #229 | ⚠️ May, but only if specific `POST` endpoint is *safe*. **Hint:** not supported by most caches. |
| `PUT` | No | Yes | No |
| `PATCH` | No | ⚠️ No, but #229 | No |
| `DELETE` | No | Yes | No |
| `OPTIONS` | Yes | Yes | No |
| `TRACE` | Yes | Yes | No |

**Note:** #227.


## SHOULD consider to design `POST` and `PATCH` idempotent {#rule-229}

In many cases it is helpful or even necessary to design `POST` and `PATCH`
*idempotent* for clients to expose conflicts and prevent resource duplicate
(a.k.a. zombie resources) or lost updates, e.g. if same resources may be
created or changed in parallel or multiple times. To design an *idempotent*
API endpoint owners should consider to apply one of the following three
patterns.

- A resource specific **conditional key** provided via `If-Match` header (#182)
  in the request. The key is in general a meta information of the resource,
  e.g. a *hash* or *version number*, often stored with it. It allows to detect
  concurrent creations and updates to ensure *idempotent* behavior (see
  #182).
- A resource specific **secondary key** provided as resource property in the
  request body. The *secondary key* is stored permanently in the resource. It
  allows to ensure *idempotent* behavior by looking up the unique secondary
  key in case of multiple independent resource creations from different
  clients (see #231).
- A client specific **idempotency key** provided via `Idempotency-Key` header
  in the request. The key is not part of the resource but stored temporarily
  pointing to the original response to ensure *idempotent* behavior when
  retrying a request (see #230).

**Note:** While **conditional key** and **secondary key** are focused on handling
concurrent requests, the **idempotency key** is focused on providing the exact
same responses, which is even a *stronger* requirement than the idempotency defined above. It can be combined with the two other patterns.

To decide, which pattern is suitable for your use case, please consult the
following table showing the major properties of each pattern:

| | Conditional Key | Secondary Key | Idempotency Key |
| --- | --- | --- | --- |
| Applicable with | `PATCH` | `POST` | `POST`/`PATCH` |
| HTTP Standard | Yes | No | No |
| Prevents duplicate (zombie) resources | Yes | Yes | No |
| Prevents concurrent lost updates | Yes | No | No |
| Supports safe retries | Yes | Yes | Yes |
| Supports exact same response | No | No | Yes |
| Can be inspected (by intermediaries) | Yes | No | Yes |
| Usable without previous `GET` | No | Yes | Yes |

**Note:** The patterns applicable to `PATCH` can be applied in the same way to
`PUT` and `DELETE` providing the same properties.

If you mainly aim to support safe retries, we suggest to apply conditional key (#182) and secondary key (#231) pattern before the idempotency key (#230) pattern.

**Note:** like for `PUT`, successful `POST` or `PATCH` returns `200` or `204` (if
the resource was updated -- with or without returning the resource), or `201`
(if resource was created). Hence, clients can differentiate successful robust
repetition from resource created server activity of idempotent `POST`.


## SHOULD use secondary key for idempotent `POST` design {#rule-231}

The most important pattern to design `POST` *idempotent* for creation is to
introduce a resource specific **secondary key** provided in the request body, to
eliminate the problem of duplicate (a.k.a zombie) resources.

The secondary key is stored permanently in the resource as *alternate key* or
*combined key* (if consisting of multiple properties) guarded by a uniqueness
constraint enforced server-side, that is visible when reading the resource.
The best and often naturally existing candidate is a *unique foreign key*, that
points to another resource having *one-on-one* relationship with the newly
created resource, e.g. a parent process identifier.

A good example here for a secondary key is the shopping cart ID in an order
resource.

**Note:** When using the secondary key pattern without `Idempotency-Key` all
subsequent retries should fail with status code `409` (conflict). We suggest
to avoid `200` here unless you make sure, that the delivered resource is the
original one implementing a well defined behavior. Using `204` without content
would be a similar well defined option.


## MAY support asynchronous request processing {#rule-253}

Typically REST APIs are designed as synchronous interfaces where all
server-side processing and state changes initiated by the call are finished
before delivering the result as response. However, in long running request
processing situations you may make use of asynchronous interface design with
multiple calls: one for initiating the asynchronous processing and subsequent
ones for accessing the processing status and/or result.

We recommend an API design that represents the asynchronous request processing
explicitly via a job resource that has a status and is different from the
actual business resource. For instance, `POST /report-jobs` returns HTTP status
code `201` to indicate successful initiation of asynchronous processing
together with the *job-id* passed in the response payload and/or via the URL of
the `Location` header. The *job-id* or `Location` URL then can be used to poll
the processing status via `GET /report-jobs/{id}` which returns HTTP status
code `200` with job status and optional report-id as response payload. Once
returned with job status `finished`, the report-id is provided and can be used
to fetch the result via `GET /reports/{id}` which returns `200` and the report
object as response payload.

Alternatively, if you do not to follow the recommended practice of providing a
separate job resource, you may use `POST /reports` returning a status code
`202` together with the `Location` header to indicate successful initiation of
the asynchronous processing. The `Location` URL is used to fetch the report via
`GET /reports/{id}` which returns either `200` and the report resource or `202`
without payload, if the asynchronous processing is still ongoing.

**Hint:** Do **not** use response code `204` or `404` instead of `202` here -- it
is misleading since neither is the processing successfully finished, nor do we
want to suggest a client failure.


## MUST define collection format of header and query parameters {#rule-154}

Header and query parameters allow to provide a collection of values, either
by providing a comma-separated list of values or by repeating the parameter
multiple times with different values as follows:

| Parameter Type | Comma-separated Values | Multiple Parameters | Standard |
| --- | --- | --- | --- |
| Header | `Header: value1,value2` | `Header: value1, Header: value2` | [RFC 9110 Section 5.3](https://tools.ietf.org/html/rfc9110#section-5.3) |
| Query | `?param=value1,value2` | `?param=value1&param=value2` | [RFC 6570 Section 3.2.8](https://tools.ietf.org/html/rfc6570#section-3.2.8) |

As OpenAPI does not support both schemas at once, an API specification must
explicitly define the collection format to guide consumers as follows:

| Parameter Type | Comma-separated Values | Multiple Parameters |
| --- | --- | --- |
| Header | `style: simple, explode: false` | not allowed (see [RFC 9110 Section 5.3](https://tools.ietf.org/html/rfc9110#section-5.3)) |
| Query | `style: form, explode: false` | `style: form, explode: true` |

When choosing the collection format, take into account the tool support,
the escaping of special characters and the maximal URL length.


## SHOULD design simple query languages using query parameters {#rule-236}

We prefer the use of query parameters to describe resource-specific query
languages for the majority of APIs because it's native to HTTP, easy to extend
and has an excellent implementation support in HTTP clients and web frameworks.

By simple query language we mean one or more name-value pairs that are combined
in one way only with `and` semantics.

Query parameters should have the following aspects specified:

- Reference to corresponding property, if any
- Value range, e.g. inclusive vs. exclusive
- Comparison semantics (equals, less than, greater than, etc)
- Implications when combined with other queries, e.g. *and* vs. *or*

How query parameters are named and used is up to individual API designers, here
are a few tips that could help to decide whether to use simple or more complex
query language:

1. Consider using simple query language when API is built to be used by others
   (external teams):

   - no additional effort/logic to form the query
   - no ambiguity in meaning of the query parameters. For example
     in `GET /items?user_id=gt:100`, is `user_id` greater than `100` or
     is `user_id` equal to `gt:100`?
   - easy to read, no learning curve

2. For internal usage or specific use case a more complex query language can be
   used (such as `price gt 10` or `price[gt]=10` or `price>10` etc.). Also
   please consider following our guidance (#237) for designing complex query
   languages with JSON.

The following examples should serve as ideas for simple query language:

### Equals

- `name=Zalando`, `creation_year=2023`, `updated_by=user1` (query elements
  based on property equality)
- `created_at=2023-09-18T12:12:00.000Z`, `age=18` (query elements
  based on logical properties)
- `color=red,green,blue,multicolored` (query elements based on multiple
  choice possibility)
  - for these type of filters, consider to use guidance (#237) to have
    smth like `filters={"color":["red","green","blue"]}`.

### Less than

- `max_length=5` -- query elements based on upper/lower bounds (`min` and `max`)
- `shorter_than=5` -- query elements using terminology specific e.g. to *length*
- `price_lower_than=50` or `price_lower_than_or_equal=50`
- `created_before=2019-07-17` or `active_until=2023-09-18T12:12:00.000Z`
  - Using terminology specific to time: *before*, *after*, *since* and *until*

### More than

- `min_length=2` -- query elements based on upper/lower bounds (`min` and `max`)
- `created_after=2019-07-17` or `modified_since=2019-07-17`
  - Using terminology specific to time: *before*, *after*, *since* and *until*
- `price_higher_than=50` or `price_equal_or_higher_than=50`

### Pagination

- `offset=10` and `limit=5` (query elements for pagination regardless
  customer sorting)
- `limit=5` and `created_after=2019-07-17` (query elements for
  keyset pagination)
  - when sorting is in place and new elements are inserted, it prevents showing
    repeated/missing results due to offset shift.

Please check conventional query parameters for pagination and sorting (#137)
and you can also find additional info in the *Pagination* section below.

We don't advocate for or against certain names because in the end APIs should
be free to choose the terminology that fits their domain the best.


## SHOULD design complex query languages using JSON {#rule-237}

Minimalistic query languages based on query parameters (#236) are suitable
for simple use cases with a small set of available filters that are combined
in one way and one way only (e.g. *and* semantics). Simple query languages are
generally preferred over complex ones.

Some APIs will have a need for sophisticated and more complex query languages.
Dominant examples are APIs around search (incl. faceting) and product catalogs.

Aspects that set those APIs apart from the rest include but are not limited to:

- Unusual high number of available filters
- Dynamic filters, due to a dynamic and extensible resource model
- Free choice of operators, e.g. `and`, `or` and `not`

APIs that qualify for a specific, complex query language are encouraged to use
nested JSON data structures and define them using OpenAPI directly. The
provides the following benefits:

- Data structures are easy to use for clients
  - No special library support necessary
  - No need for string concatenation or manual escaping
- Data structures are easy to use for servers
  - No special tokenizers needed
  - Semantics are attached to data structures rather than text tokens
- Consistent with other HTTP methods
- API is defined in OpenAPI completely
  - No external documents or grammars needed
  - Existing means are familiar to everyone

JSON-specific rules and most certainly needs to make use of the `GET`-with-body pattern.


### Example

The following JSON document should serve as an idea how a structured query
might look like.

```json
{
  "and": {
    "name": {
      "match": "Alice"
    },
    "age": {
      "or": {
        "range": {
          ">": 25,
          "<=": 50
        },
        "=": 65
      }
    }
  }
}
```

Feel free to also get some inspiration from:

- [Elastic Search: Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)
- [GraphQL: Queries](https://graphql.org/learn/queries/)


## MUST document implicit response filtering {#rule-226}

Sometimes certain collection resources or queries will not list all the
possible elements they have, but only those for which the current client is
authorized to access.

Implicit filtering could be done on:

- the collection of resources being returned on a `GET` request
- the fields returned for the detail information of the resource

In such cases, the fact that implicit filtering is applied must be documented
in the API specification's endpoint description. Consider caching aspects (#227) when implicit filtering is provided. Example:

If an employee of the company *Foo* accesses one of our business-to-business
service and performs a `GET /business-partners`, it must, for legal reasons,
not display any other business partner that is not owned or contractually
managed by her/his company. It should never see that we are doing business
also with company *Bar*.

Response as seen from a consumer working at `FOO`:

```json
{
    "items": [
        { "name": "Foo Performance" },
        { "name": "Foo Sport" },
        { "name": "Foo Signature" }
    ]
}
```

Response as seen from a consumer working at `BAR`:

```json
{
    "items": [
        { "name": "Bar Classics" },
        { "name": "Bar pour Elle" }
    ]
}
```

The API Specification should then specify something like this:

```yaml
paths:
  /business-partner:
    get:
      description: >-
        Get the list of registered business partner.
        Only the business partners to which you have access to are returned.
```
