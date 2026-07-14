# Introduction

Modern software architecture centers around decoupled microservices
that provide functionality via RESTful APIs with a JSON payload. Small
engineering teams own, deploy and operate these microservices. APIs
express most purely what systems do, and are therefore highly valuable
business assets. Designing high-quality, long-lasting APIs is essential,
especially when developing an open platform strategy that makes APIs
available for external business partners to use via third-party applications.

With this in mind, "API First" is a key engineering principle. Microservices
development begins with API specification outside the code and ideally involves
ample peer-review feedback to achieve high-quality APIs. API First encompasses a
set of quality-related standards and fosters a peer review culture including a
lightweight review procedure. Teams should follow these to ensure that APIs:

- are easy to understand and learn
- are general and abstracted from specific implementation and use cases
- are robust and easy to use
- have a common look and feel
- follow a consistent RESTful style and syntax
- are consistent with other teams' APIs and global architecture

Ideally, all APIs will look like the same author created them.


## Conventions used in these guidelines

The requirement level keywords "MUST", "MUST NOT", "REQUIRED", "SHALL",
"SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and
"OPTIONAL" used in this document (case insensitive) are to be
interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).


## About these guidelines

The purpose of these RESTful API guidelines is to define standards to
successfully establish "consistent API look and feel" quality. The API Guild
drafted and owns this document. Teams are responsible to fulfill
these guidelines during API development and are encouraged to contribute
to guideline evolution via pull requests.

These guidelines will, to some extent, remain work in progress as the
work evolves, but teams can confidently follow and trust them.

In case guidelines are changing, following rules apply:

- existing APIs don't have to be changed, but we recommend it
- clients of existing APIs have to cope with these APIs based on
outdated rules
- new APIs have to respect the current guidelines

Furthermore, keep in mind that once an API becomes publicly available, it has to
be re-reviewed and changed according to current guidelines — for the sake of
overall consistency.
