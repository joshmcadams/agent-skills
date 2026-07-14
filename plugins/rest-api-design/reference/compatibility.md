# REST Design - Compatibility


## MUST not break backward compatibility {#rule-106}

Change APIs, but keep all consumers running. Consumers usually have independent
release lifecycles, focus on stability, and avoid changes that do not provide
additional value. APIs are contracts between service providers and service
consumers that cannot be broken via unilateral decisions.

There are two techniques to change APIs without breaking them:

- follow rules for compatible extensions
- introduce new API versions and still support older versions with
  [deprecation](https://opensource.zalando.com/restful-api-guidelines/#deprecation)

We strongly encourage using compatible API extensions and discourage versioning
(see [#113](#rule-113) and [#114](#rule-114) below). The following guidelines for service providers
([#107](#rule-107)) and consumers ([#108](#rule-108)) enable us (having Postel’s Law in mind) to
make compatible changes without versioning.

**Note:** There is a difference between incompatible and breaking changes.
Incompatible changes are changes that are not covered by the compatibility
rules below. Breaking changes are incompatible changes deployed into operation,
and thereby breaking running API consumers. Usually, incompatible changes are
breaking changes when deployed into operation. However, in specific controlled
situations it is possible to deploy incompatible changes in a non-breaking way,
if no API consumer is using the affected API aspects (see also the *Deprecation*
section).

**Hint:** Please note that the compatibility guarantees are for the "on the wire"
format. Binary or source compatibility of code generated from an API specification
is not covered by these rules. If client implementations update their
generation process to a new version of the API specification, it has to be
expected that code changes are necessary.


## SHOULD prefer compatible extensions {#rule-107}

API designers should apply the following rules to evolve RESTful APIs for
services in a backward-compatible way.

In general:

- Never change the semantic of fields (e.g. changing the semantic from
  customer-number to customer-id, as both are different unique customer keys)
- Consider [#251](http-status-codes-and-errors.md#rule-251) in case a URL has to change.

The compatibility of schema changes depends on whether the input and/or output objects are defined.

For schemas used in input only:

- Add optional fields, but never mandatory fields.
- Make mandatory fields optional, but not vice-versa.
- Don't remove fields(*).
- Input fields may have (complex) constraints being validated via server-side
  business logic. Never change the validation logic to be more restrictive and
  make sure that all constraints are clearly defined in description.
- `enum` ranges can be reduced, only if the server
  continues to accept and handle old range values.
- `enum` ranges can be extended.

(*) Hint: Removing a field can be considered as a compatible change that does not break clients, in case the service still accepts and possibly ignores it if sent by the client. However, removed fields allow later adding a same-named field with different type or semantic (which is harder to catch). We therefore define field removal as non-compatible.

For schemas used in output only:

- Add (mandatory or optional) fields.
- Make optional fields mandatory, but not vice-versa.
- Don't remove fields (*).
- `enum` ranges can be reduced.
- `enum` ranges **cannot** be extended — clients may
  not be prepared to handle it.
- You should use [#112](#rule-112) for enumerations that are used for output parameters and likely to
  be extended with growing functionality. The API specification should be updated
  first before returning new values.

(*) Hint: Removing an optional field can be considered as a compatible change that does not break clients. However, removed fields allow later adding a same-named field with different type or semantic (which is harder to catch). We therefore define optional field removal as non-compatible.

For schemas used in both input and output (which is typical in
many cases), both of these rule sets combine, i.e. you can only do changes which
are allowed in both input and output.

- Add only optional, never mandatory fields.
- Don't remove any fields.
- Don't make mandatory fields optional or make optional fields mandatory.
- Input fields may have (complex) constraints being validated via server-side
  business logic. Never change the validation logic to be more restrictive and
  make sure that all constraints are clearly defined in description.
- `enum` ranges can be reduced only if the server is ready to still accept and
   handle old values. They **cannot** be extended.
- You should use [#112](#rule-112) for enumerations that are used for output parameters and likely to
  be extended with growing functionality. The API definition should be updated
  first before returning new values.

Input/Output here is from the perspective of a service implementing and
owning the API. For the rare case of APIs implemented by other services
(and consumed by the owning service), this turns around.

## SHOULD design APIs conservatively {#rule-109}

Designers of service provider APIs should be conservative and accurate in what
they accept from clients:

- Unknown input fields in payload or URL should not be ignored; servers should
  provide error feedback to clients via an HTTP 400 response code.
  Otherwise, if unexpected fields are planned to be handled in some way instead
  of being rejected, API designers **must** document clearly how unexpected
  fields are being handled for `PUT`, `POST`, and `PATCH` requests.
- Be accurate in defining input data constraints (like formats, ranges, lengths
  etc.) — and check constraints and return dedicated error information in case
  of violations.
- Prefer being more specific and restrictive (if compliant to functional
  requirements), e.g. by defining length range of strings. It may simplify
  implementation while providing freedom for further evolution as compatible
  extensions.

Not ignoring unknown input fields is a specific deviation from Postel's Law
(e.g. see also
[The Robustness Principle Reconsidered](https://cacm.acm.org/magazines/2011/8/114933-the-robustness-principle-reconsidered/fulltext)) and a strong recommendation. Servers might
want to take different approach but should be aware of the following problems
and be explicit in what is supported:

- Ignoring unknown input fields is actually not an option for `PUT`, since it
  becomes asymmetric with subsequent `GET` response and HTTP is clear about the
  `PUT` *replace* semantics and default roundtrip expectations (see
  [RFC 9110 Section 9.3.4](https://tools.ietf.org/html/rfc9110#section-9.3.4)). Note, accepting (i.e. not
  ignoring) unknown input fields and returning it in subsequent `GET` responses
  is a different situation and compliant to `PUT` semantics.
- Certain client errors cannot be recognized by servers, e.g. attribute name
  typing errors will be ignored without server error feedback. The server
  cannot differentiate between the client intentionally providing an additional
  field versus the client sending a mistakenly named field, when the client's
  actual intent was to provide an optional input field.
- Future extensions of the input data structure might be in conflict with
  already ignored fields and, hence, will not be compatible, i.e. break clients
  that already use this field but with different type.

In specific situations, where a (known) input field is not needed anymore, it
either can stay in the API specification with "not used anymore" description or
can be removed from the API specification as long as the server ignores this
specific parameter.


## MUST prepare clients to accept compatible API extensions {#rule-108}

Service clients should apply the robustness principle:

- Be conservative with API requests and data passed as input, e.g. avoid to
  exploit definition deficits like passing megabytes of strings with
  unspecified maximum length.
- Be tolerant in processing and reading data of API responses, more
  specifically service clients must be prepared for compatible API extensions
  of response data:
  - Be tolerant with unknown fields in the payload (see also Fowler’s
    ["TolerantReader"](http://martinfowler.com/bliki/TolerantReader.html) post),
    i.e. ignore new fields but do not eliminate them from payload if needed for
    subsequent `PUT` requests.
  - Be prepared that extensible enum ([#112](#rule-112)) return parameters
    may deliver new values;
    either be agnostic or provide default behavior for unknown values, and
    do not eliminate them when passed to subsequent `PUT` requests.
    (This means you cannot simply implement it by using a limited enumeration type like a Java `enum`.)
  - Be prepared to handle HTTP status codes not explicitly specified in endpoint
    definitions. Note also, that status codes are extensible. Default handling is
    how you would treat the corresponding `x00` code (see
    [RFC 9110 Section 15](https://tools.ietf.org/html/rfc9110#section-15)).
  - Follow the redirect when the server returns HTTP status code `301` (Moved
    Permanently).

The *handling compatible extensions* section describes a best practice to implement the requirements in Java.

## MUST treat OpenAPI specification as open for extension by default {#rule-111}

The OpenAPI specification is not very specific on default extensibility
of objects, and redefines JSON-Schema keywords related to extensibility, like
`additionalProperties`. Following our compatibility guidelines, OpenAPI
object definitions are considered open for extension by default as per
[Section 5.18 "additionalProperties"](http://json-schema.org/latest/json-schema-validation.html#rfc.section.5.18) of JSON-Schema.

When it comes to OpenAPI, this means an `additionalProperties` declaration
is not required to make an object definition extensible:

- API clients consuming data must not assume that objects are closed for
  extension in the absence of an `additionalProperties` declaration and must
  ignore fields sent by the server they cannot process. This allows API
  servers to evolve their data formats.
- For API servers receiving unexpected data, the situation is slightly
  different. According to [#109](#rule-109), instead of ignoring fields,
  servers *should* reject requests whose entities contain undefined fields
  in order to signal to clients that those fields would not be stored on behalf
  of the client.
  Otherwise, if unexpected fields are planned to be handled in some way instead
  of being rejected, API designers **must** document clearly how unexpected
  fields are being handled for `PUT`, `POST`, and `PATCH` requests.

API formats must not declare `additionalProperties` to be false, as this
prevents objects being extended in the future.

Note that this guideline concentrates on default extensibility and does not
exclude the use of `additionalProperties` with a schema as a value, which might
be appropriate in some circumstances, e.g. see [#216](json-guidelines.md#rule-216).


## SHOULD avoid versioning {#rule-113}

When changing your RESTful APIs, do so in a compatible way and avoid generating
additional API versions. Multiple versions can significantly complicate
understanding, testing, maintaining, evolving, operating and releasing our
systems
([supplementary reading](http://martinfowler.com/articles/enterpriseREST.html)).

If changing an API can’t be done in a compatible way, then proceed in one of
these three ways:

- create a new resource (variant) in addition to the old resource variant
- create a new service endpoint — i.e. a new application with a new API (with a
  new domain name)
- create a new API version supported in parallel with the old API by the same
  microservice

As we discourage versioning by all means because of the manifold disadvantages,
we strongly recommend to only use the first two approaches.


## MUST use URL versioning {#rule-114}

> **Adaptation:** This rule reverses the original Zalando guideline, which
> mandated *media type versioning* (the version carried in the
> `Accept`/`Content-Type` header, e.g. `application/x.zalando.cart+json;version=2`)
> and forbade URL versioning. This adaptation uses **URL versioning** to match
> common industry practice — it is easily testable in a browser, works cleanly
> with API gateways, and is highly visible. See the top-level `NOTICE` file.

When API versioning is unavoidable — i.e. when you cannot make a change in a
compatible way (see [#113](#rule-113)) — you have to design your multi-version RESTful APIs
using **URL versioning**: include the **major** version number as a path segment
placed immediately after the `/api` base path (see [#135](urls.md#rule-135)), of the form
`v<major>`.

```http
/api/v1/customers
/api/v2/customers
```

Rules for URL versioning:

- The version segment matches `v` followed by a positive integer without leading
  zeros: `v1`, `v2`, `v3`, … It contains the **major** version only.
- Minor, backward-compatible changes **MUST NOT** change the URL. Only
  incompatible (breaking) changes justify a new major version, and versioning
  itself should be avoided wherever a compatible change is possible (see [#113](#rule-113)).
- All resources belonging to one API share the same major version segment; do
  not mix versions of the same API under a single deployment.
- Because the version lives in the path, the media type stays the plain
  `application/json` (see [#172](json-guidelines.md#rule-172)) — the version is **not** encoded in the
  `Content-Type`/`Accept` header.
- When you introduce a new major version, operate it in parallel with the
  previous version and migrate consumers using the deprecation process (see the
  *Deprecation* chapter). Prefer, where feasible, the compatible alternatives in
  [#113](#rule-113) (new resource variant, or a new service) over a new URL version.

> **Note:** This versioning method applies to the resource path. Individual
> incompatible schema changes should still be avoided in favor of compatible
> extension (see [#106](#rule-106), [#107](#rule-107), [#108](#rule-108)); a new URL version is the tool of last resort
> for genuinely breaking changes.

Further reading:
[API Versioning Has No "Right Way"](https://apisyouwonthate.com/blog/api-versioning-has-no-right-way) provides an overview on different versioning
approaches to handle breaking changes without being opinionated.


## MUST NOT use media type versioning {#rule-115}

> **Adaptation:** This rule reverses the original Zalando guideline (originally
> "MUST NOT use URL versioning"). Because this adaptation versions via the URL
> (see [#114](#rule-114)), it does **not** use media type versioning. See the `NOTICE` file.

Do not encode API version information in the media type via the
`Accept`/`Content-Type` headers, e.g. avoid
`application/x.zalando.cart+json;version=2` or similar custom versioned media
types. Media type / content-negotiation versioning is less visible, harder to
exercise from a browser or simple client, and interacts awkwardly with API
gateways and caches.

Instead, carry the major version in the URL path (see [#114](#rule-114)) and keep the media
type as the standard `application/json` (see [#172](json-guidelines.md#rule-172)).


## MUST always return JSON objects as top-level data structures {#rule-110}

In a response body, you must always return a JSON object (and not e.g. an
array) as a top level data structure to support future extensibility. JSON
objects support compatible extension by additional attributes. This allows you
to easily extend your response and e.g. add pagination later, without breaking
backwards compatibility. See [#161](pagination.md#rule-161) for an example.

Maps (see [#216](json-guidelines.md#rule-216)), even though technically objects, are also forbidden as top
level data structures, since they don't support compatible, future extensions.


## SHOULD use open-ended list of values (via `examples`) for enumeration types {#rule-112}

JSON schema `enum` is per definition a closed set of values that is assumed to be
complete and not intended for extension. This means, extending the list of values of
`enum` is considered an incompatible change, and needs to be aligned with all consumers
like other incompatible changes.

To avoid these issues, we recommend to use `enum` only if

1. the API owner has full control of the enumeration values, i.e. the list of values
  does not depend on any external tool or interface, and
2. the list of values is complete with respect to any thinkable and unthinkable
  future feature.

In all other cases, where additional values are imaginable our recommendation is this:

- Use `examples` with the list of (currently known) values
- Add `[Extensible enum](https://opensource.zalando.com/restful-api-guidelines/#112).` as a standard prefix to the description.

This indicates that only the listed values are currently possible, but
consumers need to be aware that this list can be extended without notice
(see below for details).

```yaml
delivery_method:
  type: string
  examples:
    - PARCEL
    - LETTER
    - EMAIL
  description: [Extensible enum](https://opensource.zalando.com/restful-api-guidelines/#112). The chosen delivery method of the invoice.
```

See [#240](json-guidelines.md#rule-240) about enum value naming conventions – these apply here too.


**Important**:

- API consumers must be prepared for the fact that also other values can be returned
  with server responses (or be contained in consumed events), and implement a
  fallback / default behavior for unknown new values, see  [#108](#rule-108).
- API owners must take care to extend these extensible enums in a compatible way, i.e.
  not changing the semantics of already existing / documented values.
- API implementations should validate the values provided with the input payload
  and only accept values listed in `examples`.
- The list should not be reduced for inputs (that would be an incompatible change).
- Before additional values are accepted or returned, API owners should update the API description and extend
  the `examples` list, see [#107](#rule-107).

(Note that the last 3 bullet points do not apply for uses of `examples` *without* the
 "Extensible enum." prefix in the description – here any value fitting the rest
 of the schema needs to be expected.)

Note that the plural `examples` on schemas was only introduced with OpenAPI 3.1 (with an update of the JSON schema version referenced).
APIs defined with older OpenAPI versions can't use this format, and instead need to use
the `x-extensible-enum` described below↓.

### Historic Note: `x-extensible-enum`

Previously (until October 2025) this guideline recommended a proprietary
JSON schema extension `x-extensible-enum` with the same semantic.
The example above would be specified as follows:

```yaml
delivery_methods:
  type: string
  x-extensible-enum:
    - PARCEL
    - LETTER
    - EMAIL
  description: The chosen delivery method of the invoice.
```

The "important" rules above apply in an analog way for API providers and consumers using `x-extensible-enum`.

This rule originated in the time before JSON schema and OpenAPI schema
had the plural `examples` property (OpenAPI schema had singular `example`,
JSON schema had neither).

The completeness semantic would in theory allow some validation by
intermediaries (but that was rarely implemented).
It was visible in a few tools (e.g. Swagger UI), but ignored by most others.

After event messaging systems started rejecting `x-extensible-enum` in event
type definitions, we (the API guideline maintainers) revisited this rule to
recommend `examples` instead.
