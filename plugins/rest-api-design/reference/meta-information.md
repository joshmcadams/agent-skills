# REST Basics - Meta information

## MUST contain API meta information {#rule-218}

API specifications must contain the following
[OpenAPI meta information](https://spec.openapis.org/oas/latest.html#info-object):

- `#/info/title` a (unique) identifying, functional descriptive name of the API
- `#/info/version` the API specification document version following [#116](#rule-116)
- `#/info/description`  a proper description of the API
- `#/info/contact/{name,url,email}` contact info of the team owning the API specification

The following OpenAPI extension properties **must** be additionally provided:

- `#/info/x-api-id` unique identifier of the API (see rule 215 ([#215](#rule-215)))
- `#/info/x-audience` intended target audience of the API (see rule 219 ([#219](#rule-219)))


## MUST use semantic versioning {#rule-116}

OpenAPI requires the definition of API specification version via `#/info/version`.
Note, this API specification document version is distinct from the
OpenAPI Specification version (also required, e.g. `openapi: 3.1.0`),
or the API Implementation version -- see the *Terminology* section.

We expect API designers to comply to [Semantic Versioning 2.0](http://semver.org/spec/v2.0.0.html) rules `1 - 8` and `11` with the standard version format
<MAJOR>.<MINOR>.<PATCH> as follows:

- Increment the **MAJOR** version when you make incompatible API changes
after having aligned the changes with consumers,
- Increment the **MINOR** version when you add new functionality in a
backwards-compatible manner, and
- Optionally increment the **PATCH** version when you make
backwards-compatible bug fixes or editorial changes not affecting the
functionality.

**Additional Notes:**

- **Pre-release** versions ([rule 9](http://semver.org#spec-item-9)) and
**build metadata** ([rule 10](http://semver.org#spec-item-10)) must not
be used in API version information.
- While patch versions are useful for fixing typos etc, API designers
are free to decide whether they increment it or not.
- API designers should consider to use API version `0.y.z`
([rule 4](http://semver.org/#spec-item-4)) for initial API design.

Example:

```yaml
openapi: 3.1.0
info:
  version: 1.3.7
  title: Parcel Service API
  description: API for <...>
  <...>
```


## MUST provide API identifiers {#rule-215}

Each API specification must be provisioned with a globally unique and
immutable API identifier. The API identifier is defined in the `info`-block
of the OpenAPI specification and must conform to the following definition:

```yaml
/info/x-api-id:
  type: string
  format: urn
  pattern: ^[a-z0-9][a-z0-9-:.]{6,62}[a-z0-9]$
  description: |
    Mandatory globally unique and immutable API identifier. The API
    id allows to track the evolution and history of an API specification
    as a sequence of versions.
```

API specifications will evolve and any aspect of an OpenAPI specification
may change. We require API identifiers because we want  to support API clients
and providers with API lifecycle management features, like change trackability
and history or automated backward compatibility checks. The immutable API
identifier allows the identification of all API specification versions of an
API evolution. By using  API semantic version information ([#116](#rule-116)) or API publishing date ([#192](#rule-192)) as order criteria you get the **version** or
**publication history** as a sequence of API specifications.

**Note**: While it is nice to use human readable API identifiers based on
self-managed URNs, it is recommend to stick to a UUID (freshly generated when
first creating the API) to relief API designers from any urge of changing
the API identifier while evolving the API. **Do not copy an API unless you
immediately change the API identifier in it!**

Example:

```yaml
openapi: 3.1.0
info:
  x-api-id: d0184f38-b98d-11e7-9c56-68f728c1ba70
  title: Parcel Service API
  description: API for <...>
  version: 1.5.8
  <...>
```


## MUST provide API audience {#rule-219}

Each API must be classified with respect to the intended target **audience**
supposed to consume the API, to facilitate differentiated standards on APIs
for discoverability, changeability, quality of design and documentation, as
well as permission granting. We differentiate the following API audience
groups with clear organisational and legal boundaries:

- **component-internal**:
  This is often referred to as a *team internal API* or a *product internal API*.
  The API consumers with this audience are restricted to applications of the
  same **functional component** which typically represents a specific **product**
  with clear functional scope and ownership.
  All services of a functional component / product are owned by a specific dedicated owner
  and engineering team(s). Typical examples of component-internal APIs are APIs
  being used by internal helper and worker services or that support service operation.
- **business-unit-internal**:
  The API consumers with this audience are restricted to applications of a
  specific product portfolio owned by the same business unit.
- **company-internal**:
  The API consumers with this audience are restricted to applications owned
  by the business units of the same the company (e.g. a corporate group with
  multiple legal entities).
- **external-partner**:
  The API consumers with this audience are restricted to applications of
  business partners of the company owning the API and the company itself.
- **external-public**:
  APIs with this audience can be accessed by anyone with Internet access.

**Note:** a smaller audience group is intentionally included in the wider group
and thus does not need to be declared additionally.

The API audience is provided as API meta information in the `info`-block of
the OpenAPI specification and must conform to the following specification:

```yaml
/info/x-audience:
  type: string
  x-extensible-enum:
    - component-internal
    - business-unit-internal
    - company-internal
    - external-partner
    - external-public
  description: |
    Intended target audience of the API. Relevant for standards around
    quality of design and documentation, reviews, discoverability,
    changeability, and permission granting.
```

**Note:** Exactly **one audience** per API specification is allowed. For this
reason a smaller audience group is intentionally included in the wider group
and thus does not need to be declared additionally. If parts of your API have
a different target audience, we recommend to split API specifications along
the target audience — even if this creates redundancies
([rationale (internal_link)](https://apis.zalando.net/redirect/85ee93a3-7a78-4461-8bf1-08ffdaebcd18)).

Example:

```yaml
openapi: 3.1.0
info:
  x-audience: company-internal
  title: Parcel Helper Service API
  description: API for <...>
  version: 1.2.4
  <...>
```

For details and more information on audience groups see the
[API Audience narrative (internal_link)](https://apis.zalando.net/redirect/85ee93a3-7a78-4461-8bf1-08ffdaebcd18).


## MUST/SHOULD/MAY use functional naming schema {#rule-223}

Functional naming is a powerful, yet easy way to align global resources as
*host*, *permission*, and *event names* within an application landscape. It
helps to preserve uniqueness of names while giving readers meaningful context
information about the addressed component. Besides, the most important aspect
is that it allows APIs to stay stable in the case of technical and
organizational changes (for example, when a component is renamed or ownership
transfers).

A unique `functional-name` is assigned to each functional component serving an API.
It is built of the domain name of the functional group the component belongs to
and a unique short identifier for the functional component itself:

```bnf
<functional-name>      ::= <functional-domain>-<functional-component>
<functional-domain>    ::= [a-z][a-z0-9-]* -- managed functional group of components
<functional-component> ::= [a-z][a-z0-9-]* -- name of API owning functional component
```

Depending on the API audience ([#219](#rule-219)), you **must/should/may** follow the functional
naming schema for hostnames ([#224](#rule-224)) and event names ([#213](#rule-213))
(and also permission names ([#225](#rule-225), in future) as follows:

| **Functional Naming** | **Audience** |
|---|---|
| **must** | external-public, external-partner |
| **should** | company-internal, business-unit-internal |
| **may** | component-internal |

Please see the following rules for detailed functional naming patterns:
- [#224](#rule-224)
- [#213](#rule-213)


**Guidance:** You should use a functional name registry to register your
functional name before using it. This ensures global uniqueness of your
functional names (and available domains -- including subdomains) and supports
hostname DNS resolution.


## MUST follow naming convention for hostnames {#rule-224}

Hostnames in APIs must, respectively should conform to the functional naming
depending on the audience ([#219](#rule-219)) as follows (see [#223](#rule-223) for details and
`<functional-name>` definition):

```bnf
<hostname>             ::= <functional-hostname> | <application-hostname>

<functional-hostname>  ::= <functional-name>.<api-domain>
```

Where `<api-domain>` is the base domain under which your organization's APIs
are served (e.g. `apis.example.com`).
