# REST Operation


## MUST publish OpenAPI specification for APIs {#rule-192}

All service applications must publish API Specifications of their external
APIs (see API Audience definition ([#219](#rule-219))). While this is optional
for component-internal ([#219](#rule-219)) APIs, we recommend doing so to profit from
our API management infrastructure and review services.

As described in [How to publish API Specifications (internal_link)](https://cloud.docs.zalando.net/howtos/api-publishing/),
an API specification is published with the deployment of the implementing service
by copying it to the reserved `/zalando-apis` directory. The directory must only
contain *self-contained YAML files* that each describe one API (exception see [#234](#rule-234)).
**Legacy hint:** We prefer this deployment artifact-based method over the old
`.well-known/schema-discovery` service endpoint-based publishing process
(which still is supported for backward compatibility).

**Motivation:** In our dynamic and complex service infrastructure, it is important
to provide API client developers a central place with online access to the API
specifications of all running applications. As a part of the infrastructure,
the API publishing process is used to detect API specifications and to make it
discoverable via
[Sunrise (internal_link)](https://sunrise.zalando.net/apis?group=all) and the
[API Portal (internal_link)](https://apis.zalando.net/).

**Note:** APIs are discoverable only for recently running services. Hence, make sure
to always publish the API Specification as part of the service artifact deployment.


## SHOULD monitor API usage {#rule-193}

Owners of APIs used in production should monitor API service to get
information about its using clients. This information, for instance, is
useful to identify potential review partner for API changes.

Hint: A preferred way of client detection implementation is by logging
of the client-id retrieved from the OAuth token.
