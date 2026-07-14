# Changelog

> **Adaptation:** This is the **upstream Zalando** change log, preserved for
> historical context. It describes the evolution of the original guidelines and
> may reference rules as they were originally worded — including media type
> versioning (#114) and the `/api` base-path recommendation (#135). Those two
> rules have been reversed in this adaptation (URL versioning, `/api` base path);
> see the top-level `NOTICE` and the current rule text in `compatibility.md` and
> `urls.md`. Entries below are **not** a change log of this adaptation.

This change log only contains major changes and lists major changes since October 2016.

Non-major changes are editorial-only changes or minor changes of existing guidelines, e.g. adding new error code or specific example.
Major changes are changes that come with additional obligations, or even change an existing guideline obligation.
Major changes are listed as "Rule Changes" below.

**Hint:** Most recent major changes might be missing in the list since we update it
only occasionally, not with each pull request, to avoid merge commits.
Please have a look at the
[commit list in Github](https://github.com/zalando/restful-api-guidelines/commits/main)
to see a list of all changes.

## Rule Changes

- `2026-03-16`: define time-local and date-time-local formats in #238, #169, and #255. <sup>[#859](https://github.com/zalando/restful-api-guidelines/pull/859)</sup>
- `2025-11-27`: clearly separate compatibility rules for input and output schemas in #108. <sup>[#851](https://github.com/zalando/restful-api-guidelines/pull/851)</sup>
- `2025-11-27`: deprecate x-extensible-enum in favor of examples in #112. <sup>[#837](https://github.com/zalando/restful-api-guidelines/pull/837)</sup>
- `2025-11-12`: clarity about versions of API specifications in #101, #116, #192, and new terminology section. <sup>[#853](https://github.com/zalando/restful-api-guidelines/pull/853)</sup>
- `2025-10-01`: upgrade to OpenAPI 3.1 in #101. <sup>[#850](https://github.com/zalando/restful-api-guidelines/pull/850)</sup>
- `2025-10-01`: improve cursor best practices section. <sup>[#849](https://github.com/zalando/restful-api-guidelines/pull/849)</sup>
- `2025-09-18`: fix deprecation and sunset header formats in #189. <sup>[#848](https://github.com/zalando/restful-api-guidelines/pull/848)</sup>
- `2025-05-27`: clarify interaction between get-with-body and pagination in #148 and #161. <sup>[#841](https://github.com/zalando/restful-api-guidelines/pull/841)</sup>
- `2025-05-27`: allow proprietary headers for internal audience in transactions checkout in #183. <sup>[#843](https://github.com/zalando/restful-api-guidelines/pull/843)</sup>
- `2024-06-27`: Clarified usage of `x-extensible-enum` for events in #112. <sup>[#807](https://github.com/zalando/restful-api-guidelines/pull/807)</sup>
- `2024-06-25`: Relaxed naming convention for date/time properties in #235. <sup>[#811](https://github.com/zalando/restful-api-guidelines/pull/811)</sup>
- `2024-06-11`: Linked #127 (duration / period) from #238. <sup>[#810](https://github.com/zalando/restful-api-guidelines/pull/810)</sup>
- `2024-05-06`: Added new rule #255 on selecting appropriate date or date-time format. <sup>[#808](https://github.com/zalando/restful-api-guidelines/pull/808)</sup>
- `2024-04-16`: Removed sort example from simple query language in #236. Enhanced clarity for 'uid' usage and permission naming convention in #105 and #225. <sup>[#804](https://github.com/zalando/restful-api-guidelines/pull/804)</sup> <sup>[#801](https://github.com/zalando/restful-api-guidelines/pull/801)</sup>
- `2024-03-21`: Added best practices section for #108 on handling compatible API extensions. <sup>[#799](https://github.com/zalando/restful-api-guidelines/pull/799)</sup>
- `2024-03-05`: Updated security section about uid scope in #105. <sup>[#794](https://github.com/zalando/restful-api-guidelines/pull/794)</sup>
- `2024-02-29`: Improved guidance on `POST` usage in #148. <sup>[#791](https://github.com/zalando/restful-api-guidelines/pull/791)</sup>
- `2024-02-21`: Fixed discrepancy between #109 and #111 regarding handling of unexpected fields. <sup>[#793](https://github.com/zalando/restful-api-guidelines/pull/793)</sup>
- `2023-12-12`: Improved response code guidance in #150. <sup>[#789](https://github.com/zalando/restful-api-guidelines/pull/789)</sup>
- `2023-11-22`: Added new rule #253 for supporting asynchronous request processing. <sup>[#787](https://github.com/zalando/restful-api-guidelines/pull/787)</sup>
- `2023-07-21`: Improved guidance on total counts in #254. <sup>[#731](https://github.com/zalando/restful-api-guidelines/pull/731)</sup>
- `2023-05-12`: Added new rule #251 recommending not to use redirection codes. <sup>[#762](https://github.com/zalando/restful-api-guidelines/pull/762)</sup> <sup>[#781](https://github.com/zalando/restful-api-guidelines/pull/781)</sup>
- `2023-05-08`: Improved guidance on sanitizing JSON payload in #250. <sup>[#759](https://github.com/zalando/restful-api-guidelines/pull/759)</sup>
- `2023-04-18`: Added new rule #252 recommending to design single resource schema for reading and writing. Added exception for partner IAM to #104. <sup>[#764](https://github.com/zalando/restful-api-guidelines/pull/764)</sup> <sup>[#767](https://github.com/zalando/restful-api-guidelines/pull/767)</sup>
- `2022-12-20`: Clarify that event consumers must be robust against duplicates in #214. <sup>[#749](https://github.com/zalando/restful-api-guidelines/pull/749)</sup>
- `2022-10-18`: Add X-Zalando-Customer to list of proprietary headers in #183. <sup>[#743](https://github.com/zalando/restful-api-guidelines/pull/743)</sup>
- `2022-09-21`: Clarify that functional naming schema in #223 is a **MUST**/**SHOULD**/**MAY** rule. <sup>[#740](https://github.com/zalando/restful-api-guidelines/pull/740)</sup>
- `2022-07-26`: Improve guidance for return code usage for (robust) create operations in #148. <sup>[#735](https://github.com/zalando/restful-api-guidelines/pull/735)</sup>
- `2022-07-21`: Improve format and time interval guidance in #238 and #127. <sup>[#733](https://github.com/zalando/restful-api-guidelines/pull/733)</sup>
- `2022-05-24`: Define `next` page link as optional in the *pagination fields* section. <sup>[#726](https://github.com/zalando/restful-api-guidelines/pull/726)</sup>
- `2022-04-19`: Change #200 from **MUST** to **SHOULD** avoid providing sensitive data with events. <sup>[#723](https://github.com/zalando/restful-api-guidelines/pull/723)</sup>
- `2022-03-22`: More clarity about when error specification definitions can be omitted in #151. More clarity around HTTP status codes in #243. More clarity for avoiding null for boolean fields in #122. <sup>[#715](https://github.com/zalando/restful-api-guidelines/pull/715)</sup> <sup>[#720](https://github.com/zalando/restful-api-guidelines/pull/720)</sup> <sup>[#721](https://github.com/zalando/restful-api-guidelines/pull/721)</sup>
- `2022-01-26`: Exclude 'type' from common field names in #235. Improve clarity around usage of extensible enums in #112. <sup>[#714](https://github.com/zalando/restful-api-guidelines/pull/714)</sup> <sup>[#717](https://github.com/zalando/restful-api-guidelines/pull/717)</sup>
- `2021-12-22`: Clarify that event id must not change in retry situations when producers #211. <sup>[#694](https://github.com/zalando/restful-api-guidelines/pull/694)</sup>
- `2021-12-09`: Improve clarity on `PATCH` and `PUT` usage in rule #148. Only use codes registered via IANA in rule #243. <sup>[cb0624b](https://github.com/zalando/restful-api-guidelines/commit/cb0624b2bc128b32fba0f78dd24f1a67d5e62766)</sup>
- `2021-12-09`: event id must not change in retry situations when producers #211.
- `2021-11-24`: restructuring of the document and some rules.
- `2021-10-18`: new rule #244.
- `2021-10-12`: improve clarity on `PATCH` usage in rule #148.
- `2021-08-24`: improve clarity on `PUT` usage in rule #148.
- `2021-08-24`: only use codes registered via IANA in rule #243.
- `2021-08-17`: update formats per OpenAPI 3.1 in #238.
- `2021-06-22`: #238 changed from **SHOULD** to **MUST**; consistency for rules around standards for data.
- `2021-06-03`: #104 with clear distinction of OpenAPI security schemes, favoring `bearer` to `oauth2`.
- `2021-06-01`: resolve uncertainties around 'occurred_at' semantics of event metadata.
- `2021-05-25`: #172 with API endpoint versioning (#114) as only custom media type usage exception.
- `2021-05-05`: define usage on resource-ids in `PUT` and `POST` in #148.
- `2021-04-29`: improve clarity of #133.
- `2021-03-19`: clarity on #167.
- `2021-03-15`: #242 changed from **SHOULD** to **MUST**; improve clarity around event ordering (#203).
- `2021-03-19`: best practice section *cursor-based pagination*
- `2021-02-16`: define how to reference models outside the api in #234.
- `2021-02-15`: improve guideline #176 -- clients must be prepared to not receive problem return objects.
- `2021-01-19`: more details for `GET with body` and `DELETE with body` (#148).
- `2020-09-29`: include models for headers to be included by reference in API definitions (#183)
- `2020-09-08`: add exception for legacy host names to #224
- `2020-08-25`: change #240 from **MUST** to **SHOULD**, explain exceptions
- `2020-08-25`: add exception for `self` to #143 and #134.
- `2020-08-24`: change "**MUST** avoid trailing slashes" to #136.
- `2020-08-20`: change #183 from **MUST** to **SHOULD**, mention gateway-specific headers (which are not part of the public API).
- `2020-06-30`: add details to #114
- `2020-05-19`: new sections about DELETE with query parameters and `DELETE with body` in #148.
- `2020-02-06`: new rule #241
- `2020-02-05`: add Sunset header, clarify deprecation procedure (#185, #186, #187, #188, #189, #190, #191)
- `2020-01-21`: new rule #240 (as MUST, changed later to SHOULD)
- `2020-01-15`: change "Warning" to "Deprecation" header in #189, #190.
- `2019-10-10`: remove never-implemented rule "**MUST** Permissions on events must correspond to API permissions"
- `2019-09-10`: remove duplicated rule "**MAY** Standards could be used for Language, Country and Currency", upgrade #170 from **MAY** to **SHOULD**.
- `2019-08-29`: new rule #239, extend #167 pointing to [RFC-7493](https://tools.ietf.org/html/rfc7493)
- `2019-08-29`: new rules #236, #237
- `2019-07-30`: new rule #238
- `2019-07-30`: change #173 from **SHOULD** to **MUST**
- `2019-07-30`: change "**SHOULD** Null values should have their fields removed to" #123.
- `2019-07-25`: new rule #235.
- `2019-07-18`: improved cursor guideline for `GET with body`.
- `2019-06-25`: change #154 from **SHOULD** to **MUST**, use OpenAPI 3 syntax
- `2019-06-13`: remove `X-App-Domain` from #183.
- `2019-05-17`: add `X-Mobile-Advertising-Id` to #183.
- `2019-04-09` New rule #234
- `2019-02-19`: New rule #233 extracted + expanded from #183.
- `2019-01-24:` Improve guidance on caching (#149, #227).
- `2019-01-21:` Improve guidance on idempotency, introduce idempotency-key (#229, #231).
- `2019-01-16`: Change #135 from **MAY** to **SHOULD NOT**
- `2018-10-19`: Add `ordering_key_field` to event type definition schema (#197, #203)
- `2018-09-28`: New rule #228
- `2018-09-13`: replaced OpenAPI 2.0 syntax with OpenAPI 3.0 in the example snippets
- `2018-08-10`: New rule #226
- `2018-07-12`: Add `audience` field to event type definition (#197)
- `2018-06-11:` Introduced new naming guidelines for host, permission, and event names.
- `2018-01-10:` Moved meta information related aspects into new chapter *Meta Information*.
- `2018-01-09:` Changed publication requirements for API specifications (#192).
- `2017-12-07:` Added best practices section including discussion about optimistic locking approaches.
- `2017-11-28:` Changed OAuth flow example from password to client credentials in the *Security* section.
- `2017-11-22:` Updated description of X-Tenant-ID header field
- `2017-08-22:` Migration to Asciidoc
- `2017-07-20:` Be more precise on client vs. server obligations for compatible API extensions.
- `2017-06-06:` Made money object guideline clearer.
- `2017-05-17:` Added guideline on query parameter collection format.
- `2017-05-10:` Added the convention of using RFC2119 to describe guideline levels, and replaced `book.could` with `book.may`.
- `2017-03-30:` Added rule that permissions on resources in events must correspond to permissions on API resources
- `2017-03-30:` Added rule that APIs should be modelled around business processes
- `2017-02-28:` Extended information about how to reference sub-resources and the usage of composite identifiers in the #143
part.
- `2017-02-22:` Added guidance for conditional requests with If-Match/If-None-Match
- `2017-02-02:` Added guideline for batch and bulk request
- `2017-02-01:` #180
- `2017-01-18:` Removed "Avoid Javascript Keywords" rule
- `2017-01-05:` Clarification on the usage of the term "REST/RESTful"
- `2016-12-07:` Introduced "API as a Product" principle
- `2016-12-06:` New guideline: "Should Only Use UUIDs If Necessary"
- `2016-12-04:` Changed OAuth flow example from implicit to password in the *Security* section.
- `2016-10-13:` #172
- `2016-10-10:` Introduced the changelog. From now on all rule changes on API guidelines will be recorded here.
