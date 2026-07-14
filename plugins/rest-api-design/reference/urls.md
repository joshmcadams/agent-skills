# REST Basics - URLs

Guidelines for naming and designing resource paths and query parameters.


## SHOULD use /api as base path {#rule-135}

> **Adaptation:** This rule reverses the original Zalando guideline, which
> recommended *against* an `/api` base path (serving resources from the root
> `/`). This adaptation uses an **`/api` base path** to match common industry
> practice, where API traffic shares a domain with a frontend and is routed by
> an `/api` prefix (e.g. `example.com/api/v1/users`). See the `NOTICE` file.

Resources provided by a service should be made available under an `/api` base
path, followed by the major version segment (see [#114](#rule-114)). The canonical shape of
a resource path is therefore:

```http
/api/v1/{resources}
```

This makes API traffic easy to route via a gateway or reverse proxy when the API
shares a domain with other content (e.g. a frontend application), and keeps API
endpoints clearly distinguished from non-API routes.

If the service should also support non-public, internal APIs
— for specific operational support functions, for example — we encourage
you to maintain two different API specifications and provide
API audience ([#219](#rule-219)).

The concrete host (and, if applicable, any additional path prefix) is part of
deployment variant configuration and has to be declared in the
[server object](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.2.md#server-object),
e.g. `https://{host}/api`.


## MUST pluralize resource names {#rule-134}

Usually, a collection of resource instances is provided (at least the API
should be ready here). The special case of a *resource singleton* must
be modeled as a collection with cardinality 1 including definition of
`maxItems` = `minItems` = 1 for the returned `array` structure
to make the cardinality constraint explicit.

**Exception:** the *pseudo identifier* `self` used to specify a resource endpoint
where the resource identifier is provided by authorization information (see [#143](#rule-143)).


## MUST use URL-friendly resource identifiers {#rule-228}

To simplify encoding of resource IDs in URLs they must match the regex `[a-zA-Z0-9:._\-/]*`.
Resource IDs only consist of ASCII strings using letters, numbers, underscore, minus, colon,
period, and - on rare occasions - slash.

**Note:** slashes are only allowed to build and signal resource identifiers
consisting of compound keys ([#241](#rule-241)).

**Note:** to prevent ambiguities of unnormalized paths ([#136](#rule-136)) resource
identifiers must never be empty. Consequently, support of empty strings for
path parameters is forbidden.


## MUST use kebab-case for path segments {#rule-129}

Path segments are restricted to ASCII kebab-case strings matching regex `^[a-z][a-z\-0-9]*$`.
The first character must be a lower case letter, and subsequent
characters can be a letter, or a dash(`-`), or a number.

Example:

```http
/shipment-orders/{shipment-order-id}
```

**Hint:** kebab-case applies to concrete path segments and not necessarily the names of path parameters.


## MUST use normalized paths without empty path segments and trailing slashes {#rule-136}

You must not specify paths with duplicate or trailing slashes, e.g.
`/customers//addresses` or `/customers/`. As a consequence, you must also not
specify or use path variables with empty string values.

**Note:** Non standard paths have no clear semantics. As a result, behavior
for non standard paths varies between different HTTP infrastructure components
and libraries. This may leads to ambiguous and unexpected results during
request handling and monitoring.

We recommend to implement services robust against clients not following this
rule. All services **should** [normalize](https://en.wikipedia.org/wiki/URI_normalization)
request paths before processing by removing duplicate and trailing slashes.
Hence, the following requests should refer to the same resource:

```http
GET /orders/{order-id}
GET /orders/{order-id}/
GET /orders//{order-id}
```

**Note:** path normalization is not supported by all framework out-of-the-box.
Services are required to support at least the normalized path while rejecting
all alternatives paths, if failing to deliver the same resource.


## MUST keep URLs verb-free {#rule-141}

The API describes resources, so the only place where actions should appear is
in the HTTP methods. In URLs, use only nouns. Instead of thinking of actions
(verbs), it's often helpful to think about putting a message in a letter box:
e.g., instead of having the verb *cancel* in the url, think of sending a
message to cancel an order to the *cancellations* letter box on the server
side.


## MUST avoid actions — think about resources {#rule-138}

REST is all about your resources, so consider the domain entities that take
part in web service interaction, and aim to model your API around these using
the standard HTTP methods as operation indicators. For instance, if an
application has to lock articles explicitly so that only one user may edit
them, create an article lock with `PUT` or `POST` instead of using a lock
action.

Request:

```http
PUT /article-locks/{article-id}
```

The added benefit is that you already have a service for browsing and filtering
article locks.


## SHOULD define *useful* resources {#rule-140}

As a rule of thumb resources should be defined to cover 90% of all its client's
use cases. A *useful* resource should contain as much information as necessary,
but as little as possible. A great way to support the last 10% is to allow
clients to specify their needs for more/less information by supporting
filtering and embedding ([#157](#rule-157)).


## MUST use domain-specific resource names {#rule-142}

API resources represent elements of the application’s domain model. Using
domain-specific nomenclature for resource names helps developers to understand
the functionality and basic semantics of your resources. It also reduces the
need for further documentation outside the API specification. For example,
"sales-order-items" is superior to "order-items" in that it clearly indicates
which business object it represents. Along these lines, "items" is too general.


## SHOULD model complete business processes {#rule-139}

An API should contain the complete business processes containing all resources
representing the process. This enables clients to understand the business
process, foster a consistent design of the business process, allow for
synergies from description and implementation perspective, and eliminates
implicit invisible dependencies between APIs.

In addition, it prevents services from being designed as thin wrappers around
databases, which normally tends to shift business logic to the clients.


## MUST identify resources and sub-resources via path segments {#rule-143}

Some API resources may contain or reference sub-resources. Embedded
sub-resources, which are not top-level resources, are parts of a higher-level
resource and cannot be used outside of its scope. Sub-resources should be
referenced by their name and identifier in the path segments as follows:

```http
/resources/{resource-id}/sub-resources/{sub-resource-id}
```

In order to improve the consumer experience, you should aim for intuitively
understandable URLs, where each sub-path is a valid reference to a resource or
a set of resources. For instance, if
`/partners/{partner-id}/addresses/{address-id}` is valid, then, in principle,
also `/partners/{partner-id}/addresses`, `/partners/{partner-id}` and
`/partners` must be valid. Examples of concrete url paths:

```http
/shopping-carts/de:1681e6b88ec1/items/1
/shopping-carts/de:1681e6b88ec1
/content/images/9cacb4d8
/content/images
```

**Note:** resource identifiers may be build of multiple other resource
identifiers (see [#241](#rule-241)).

**Exception:** In some situations the resource identifier is not passed as a
path segment but  via the authorization information, e.g. an authorization
token or session cookie. Here, it is reasonable to use **`self`** as
*pseudo-identifier* path segment. For instance, you may define `/employees/self`
or `/employees/self/personal-details` as resource paths --  and may additionally
define endpoints that support identifier passing in the resource path, like
define `/employees/{empl-id}` or `/employees/{empl-id}/personal-details`.


## MAY expose compound keys as resource identifiers {#rule-241}

If a resource is best identified by a *compound key* consisting of multiple
other resource identifiers, it is allowed to reuse the compound key in its
natural form containing slashes instead of *technical* resource identifier in
the resource path without violating the above rule [#143](#rule-143) as follows:

```http
/resources/{compound-key-1}[delim-1]...[delim-n-1]{compound-key-n}
```

Example paths:

```http
/shopping-carts/{country}/{session-id}
/shopping-carts/{country}/{session-id}/items/{item-id}
/api-specifications/{docker-image-id}/apis/{path}/{file-name}
/api-specifications/{repository-name}/{artifact-name}:{tag}
/article-size-advices/{sku}/{sales-channel}
```

**Note**: Exposing a compound key as described above limits ability to
evolve the structure of the resource identifier as it is no longer opaque.

To compensate for this drawback, APIs must apply a compound key abstraction
consistently in all requests and responses parameters and attributes allowing
consumers to treat these as *technical resource identifier* replacement. The
use of independent compound key components must be limited to search and
creation requests, as follows:

```http
# compound key components passed as independent search query parameters
GET /article-size-advices?skus=sku-1,sku-2&sales_channel_id=sid-1
=> { "items": [{ "id": "id-1", ...  },{ "id": "id-2", ...  }] }

# opaque technical resource identifier passed as path parameter
GET /article-size-advices/id-1
=> { "id": "id-1", "sku": "sku-1", "sales_channel_id": "sid-1", "size": ... }

# compound key components passed as mandatory request fields
POST /article-size-advices { "sku": "sku-1", "sales_channel_id": "sid-1", "size": ... }
=> { "id": "id-1", "sku": "sku-1", "sales_channel_id": "sid-1", "size": ... }
```

Where `id-1` is representing the opaque provision of the compound key
`sku-1/sid-1` as technical resource identifier.

**Remark:** A compound key component may itself be used as another resource
identifier providing another resource endpoint, e.g `/article-size-advices/{sku}`.


## MAY consider using (non-) nested URLs {#rule-145}

If a sub-resource is only accessible via its parent resource and may not exist
without parent resource, consider using a nested URL structure, for instance:

```http
/shoping-carts/de/1681e6b88ec1/cart-items/1
```

However, if the resource can be accessed directly via its unique id, then the
API should expose it as a top-level resource. For example, customer has a
collection for sales orders; however, sales orders have globally unique id and
some services may choose to access the orders directly, for instance:

```http
/customers/1637asikzec1
/sales-orders/5273gh3k525a
```


## SHOULD limit number of resource types {#rule-146}

To keep maintenance and service evolution manageable, we should follow
"functional segmentation" and "separation of concern" design principles and do
not mix different business functionalities in same API. In practice
this means that the number of resource types exposed via an API should be
limited. In this context a resource type is defined as a set of highly related
resources such as a collection, its members and any direct sub-resources.

For example, the resources below would be counted as three resource types, one
for customers, one for the addresses, and one for the customers' related
addresses:

```http
/customers
/customers/{id}
/customers/{id}/preferences
/customers/{id}/addresses
/customers/{id}/addresses/{addr}
/addresses
/addresses/{addr}
```

Note that:

- We consider `/customers/{id}/preferences` part of the `/customers` resource
  type because it has a one-to-one relation to the customer without an
  additional identifier.
- We consider `/customers` and `/customers/{id}/addresses` as separate resource
  types because `/customers/{id}/addresses/{addr}` also exists with an
  additional identifier for the address.
- We consider `/addresses` and `/customers/{id}/addresses` as separate resource
  types because there's no reliable way to be sure they are the same.

Given this definition, our experience is that well defined APIs involve no more
than 4 to 8 resource types. There may be exceptions with more complex business
domains that require more resources, but you should first check if you can
split them into separate subdomains with distinct APIs.

Nevertheless one API should hold all necessary resources to model complete
business processes helping clients to understand these flows.


## SHOULD limit number of sub-resource levels {#rule-147}

There are main resources (with root url paths) and sub-resources (or *nested*
resources with non-root urls paths). Use sub-resources if their life cycle is
(loosely) coupled to the main resource, i.e. the main resource works as
collection resource of the subresource entities. You should use <= 3
sub-resource (nesting) levels -- more levels increase API complexity and url
path length. (Remember, some popular web browsers do not support URLs of more
than 2000 characters.)


## MUST use snake_case (never camelCase) for query parameters {#rule-130}

See also [#118](#rule-118).

## MUST stick to conventional query parameters {#rule-137}

If you provide query support for searching, sorting, filtering, and
paginating, you must stick to the following naming conventions:

- `q`: default query parameter, e.g. used by browser tab completion;
  should have an entity specific alias, e.g. sku.
- `sort`: comma-separated list of fields (as defined by [#154](#rule-154)) to
  define the sort order. To indicate sorting direction, fields may be prefixed
  with `+` (ascending) or `-` (descending), e.g. /sales-orders?sort=+id.
- `fields`: field name expression to retrieve only a subset of fields
  of a resource. See [#157](#rule-157) below.
- `embed`: field name expression to expand or embedded sub-entities,
  e.g. inside of an article entity, expand silhouette code into the silhouette
  object. Implementing `embed` correctly is difficult, so do it with care.
  See [#158](#rule-158) below.
- `offset`: numeric offset of the first element provided on a page
  representing a collection request. See the *Pagination* section below.
- `cursor`: an opaque pointer to a page, never to be inspected or
  constructed by clients. It usually (encrypted) encodes the page position,
  i.e. the identifier of the first or last page element, the pagination
  direction, and the applied query filters to recreate the collection. See
  the *cursor-based pagination* section or the *Pagination* section below.
- `limit`: client suggested limit to restrict the number of entries on
  a page. See the *Pagination* section below.
