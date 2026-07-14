# REST Design - Deprecation

Sometimes it is necessary to phase out an API endpoint, an API version, or an
API feature, e.g. if a field or parameter is no longer supported or a whole
business functionality behind an endpoint is supposed to be shut down. As long
as the API endpoints and features are still used by consumers these shut downs
are breaking changes and not allowed. To progress the following deprecation
rules have to be applied to make sure that the necessary consumer changes and
actions are well communicated and aligned using *deprecation* and *sunset*
dates.


## MUST reflect deprecation in API specifications {#rule-187}

The API deprecation must be part of the API specification.

If an API endpoint (operation object), an input argument (parameter object),
an in/out data object (schema object), or on a more fine grained level, a
schema attribute or property should be deprecated, the producers must set
`deprecated: true` for the affected element and add further explanation to the
`description` section of the API specification. If a future shut down is
planned, the producer must provide a sunset date and document in details
what consumers should use instead and how to migrate.


## MUST obtain approval of clients before API shut down {#rule-185}

Before shutting down an API, version of an API, or API feature the producer
must make sure, that all clients have given their consent on a sunset date.
Producers should help consumers to migrate to a potential new API or API
feature by providing a migration manual and clearly state the time line for
replacement availability and sunset (see also [#189](#rule-189)). Once all clients of
a sunset API feature are migrated, the producer may shut down the deprecated
API feature.


## MUST collect external partner consent on deprecation time span {#rule-186}

If the API is consumed by any external partner, the API owner must define a
reasonable time span that the API will be maintained after the producer has
announced deprecation. All external partners must state consent with this
after-deprecation-life-span, i.e. the minimum time span between official
deprecation and first possible sunset, **before** they are allowed to use the
API.


## MUST monitor usage of deprecated API scheduled for sunset {#rule-188}

Owners of an API, API version, or API feature used in production that is
scheduled for sunset must monitor the usage of the sunset API, API version, or
API feature in order to observe migration progress and avoid uncontrolled
breaking effects on ongoing consumers. See also [#193](api-operation.md#rule-193).


## SHOULD add `Deprecation` and `Sunset` header to responses {#rule-189}

During the deprecation phase, the producer should add a
`Deprecation: <timestamp>` (see [RFC 9745 section 2](https://tools.ietf.org/html/rfc9745#section-2))
and - if also planned - a `Sunset: <date-time>` (see [RFC 8594 section 3](https://tools.ietf.org/html/rfc8594#section-3)) header on each response affected by a deprecated element (see
[#187](#rule-187)).

**Note:** We explicitly discourage the use of `Link` relation headers using
`rel="sunset|deprecation"` as defined in [RFC 9745 section 3](https://tools.ietf.org/html/rfc9745#section-3) and [RFC 8594 section 6](https://tools.ietf.org/html/rfc8594#section-6) as these do not provide a
significant benefit over the documentation provided via the API specification.

The `Deprecation` header can either be set to `true` - if a feature is retired
-, or carry a deprecation timestamp [RFC-9651 section 3.3.7](https://tools.ietf.org/html/rfc9651#section-3.3.7), at which a replacement will become/became available and consumers must
not on-board any longer (see [#191](#rule-191)). The optional `Sunset` date carries
the information when consumers latest have to stop using a feature. The sunset
date should always offer an eligible time interval for switching to a
replacement feature. Note, the deprecation and sunset headers use different time formats due to historic reasons:

```txt
Deprecation: @1758095283
Sunset: Wed, 31 Dec 2025 23:59:59 GMT
```

If multiple elements are deprecated the `Deprecation` and `Sunset` headers are
expected to be set to the earliest time stamp to reflect the shortest interval
at which consumers are expected to get active. The `Deprecation` and `Sunset`
headers can be defined as follows in the API specification (see also the
default definition below):

```yaml
components:
  parameters|headers:
    Deprecation:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/Deprecation'
    Sunset:
      $ref: 'https://opensource.zalando.com/restful-api-guidelines/models/headers-1.0.0.yaml#/Sunset'
```

```yaml
components:
  headers:
    Deprecation:
        required: false
        description: |-
          The **Deprecation** response header (see [RFC 9745][rfc-9745]) announces
          an upcoming deprecation of a feature or resource. The deprecation value is
          either a timestamp as defined in [RFC 9651 Section 3.3.7][rfc-9651-3.3.7]
          or `true` - if the feature is already deprecated.

          The scope of the distinct deprecated feature or resource must be derived
          from the API spec. If the timestamp points to the past, the API spec also
          contains a migration advise.

          Clients should monitor the usage of **Deprecation** headers and notify
          about them in time, so that migration measures can be planned and executed
          timely (see also [API Guideline - Deprecation][api]).

          [rfc-9745]: <https://tools.ietf.org/html/rfc9745>
          [rfc-9651-3.3.7]: <https://tools.ietf.org/html/rfc9651#section-3.3.7>
          [api]: <https://opensource.zalando.com/restful-api-guidelines/#deprecation>
        schema:
          type: string
          format: date-timestamp
        example: "@1758093035"

    Sunset:
        required: false
        description: |-
          The **Sunset** response header (see [RFC 8594][rfc-8594]) communicates the
          point in time at which the feature or resource becomes unresponsive. The
          sunset value is providing a timestamp as defined in [RFC 9110 Section
          5.6.7][rfc-9110-5.6.7] usually pointing to the future. If a sunset value
          points to the past, the feature or resource must be expected to become
          unavailable at any time.

          Clients should monitor the usage of **Sunset** headers and warn/alert about
          them before the sunset time has come to take counter measures, e.g. prepare
          a client shutdown or migration (see also [API Guideline -
          Deprecation][api-deprecation]).

          [rfc-8594]: <https://tools.ietf.org/html/rfc8594>
          [rfc-9110-5.6.7]: <https://tools.ietf.org/html/rfc9110#section-5.6.7>
          [api-deprecation]: <https://opensource.zalando.com/restful-api-guidelines/#deprecation>
        schema:
          type: string
          format: http-date
        example: "Wed, 31 Dec 2025 23:59:59 GMT"
```

**Note:** adding the `Deprecation` and `Sunset` header is not sufficient to gain
client consent to shut down an API or feature.

**Hint:** In earlier guideline versions, we used the `Warning` header to provide
the deprecation info to clients. However, `Warning` header has a less specific
semantics, will be obsolete with
[draft: RFC HTTP Caching](https://tools.ietf.org/html/draft-ietf-httpbis-cache-06), and our syntax was not compliant with [RFC 9111 Section 5.5 "Warning"](https://tools.ietf.org/html/rfc9111#section-5.5).


## SHOULD add monitoring for `Deprecation` and `Sunset` header {#rule-190}

Clients should monitor the `Deprecation` and `Sunset` headers in HTTP responses
to get information about future sunset of APIs and API features (see [#189](#rule-189)).
We recommend that client owners build alerts on this monitoring information to
ensure alignment with service owners on required migration task.

**Hint:** In earlier guideline versions, we used the `Warning` header to provide
the deprecation info (see hint in [#189](#rule-189)).


## MUST not start using deprecated APIs {#rule-191}

Clients must not start using deprecated APIs, API versions, or API features.
