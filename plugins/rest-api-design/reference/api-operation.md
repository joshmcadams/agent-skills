# REST Operation


## MUST publish OpenAPI specification for APIs {#rule-192}

All service applications must publish API specifications of their external
APIs (see API Audience definition ([#219](#rule-219))). While this is optional
for component-internal ([#219](#rule-219)) APIs, we recommend doing so to benefit from
API management infrastructure and review services.

API specifications should be published alongside the deployment of the
implementing service, typically by copying a self-contained YAML file to a
well-known location. The published directory must only contain *self-contained
YAML files* that each describe one API (exception see [#234](#rule-234)).

**Motivation:** In a dynamic and complex service infrastructure, it is important
to provide API client developers a central place with online access to the API
specifications of all running applications. The API publishing process enables
detection of API specifications and makes them discoverable through an API
portal or developer hub.

**Note:** APIs are discoverable only for recently running services. Hence, make
sure to always publish the API specification as part of the service artifact
deployment.


## SHOULD monitor API usage {#rule-193}

Owners of APIs used in production should monitor API service to get
information about its using clients. This information, for instance, is
useful to identify potential review partner for API changes.

Hint: A preferred way of client detection implementation is by logging
of the client-id retrieved from the OAuth token.
