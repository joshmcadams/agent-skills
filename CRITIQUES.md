# CRITIQUES.md — Skills Review

Findings from a review of the 21 skills in `plugins/rest-api-design/skills/`,
conducted after the second agent's additions. Each item is a concrete issue
with a suggested fix.

---

## 1. `${CLAUDE_PLUGIN_ROOT}` re-introduced after removal

**Impact:** High. The skills were previously converted to relative paths
(`../../reference/...`) in commit `5dab25d`. The other agent's update
re-introduced `${CLAUDE_PLUGIN_ROOT}` everywhere — adaptation blocks,
inline `$ref` examples, and Reference sections — with the fallback note:
"(If `${CLAUDE_PLUGIN_ROOT}` is not defined in your environment, the plugin
root is the directory two levels above this SKILL.md.)"

The fallback is helpful, but the skills should settle on one convention.
Option A: restore relative paths and keep the fallback note. Option B: accept
the env-var convention and ensure every consumer documents how to set it.

**Files:** All 21 SKILL.md files, `deprecate/SKILL.md` and
`modify-schema/SKILL.md` in particular (inline `$ref` examples referencing
`${CLAUDE_PLUGIN_ROOT}`).

## 2. `api-security` tagged `knowledge` but has a builder mode

**Impact:** Medium. The frontmatter says `type: knowledge`, but the skill now
includes a full "Applying it to a spec (builder mode)" section (5 steps: detect
artifact, add securitySchemes, assign scopes, apply least privilege, check URLs
for secrets). An agent scanning the `type` field alone would classify this as
advisory-only, missing the builder capability.

**Suggested fix:** Either (a) change `type` to `builder` since it now produces
edits, with a note that it advises by default, or (b) introduce a
`type: hybrid` convention for skills that do both.

**File:** `api-security/SKILL.md`

## 3. `api-security` carries pre-generalization phrasing

**Impact:** Medium. Rule 1 says: "Most internal, service-to-service APIs use
JWT bearer tokens from the platform IAM token service." The "platform IAM
token service" is Zalando-specific framing that survived the reference-doc
generalization. `security.md` was one of the files generalized (#9), but the
skill wasn't updated to match.

**Suggested fix:** Replace with: "Most internal, service-to-service APIs use
JWT bearer tokens (`Authorization: Bearer <token>`, RFC 6750)."

**File:** `api-security/SKILL.md`, rule 1.

## 4. `json-conventions` recommends legacy `x-extensible-enum`

**Impact:** Medium. The skill says "Prefer extensible enums (`x-extensible-enum`)"
but the reference docs (`compatibility.md` #112) were updated to prefer the
OpenAPI-native `examples` list with an `[Extensible enum]` description prefix.
`x-extensible-enum` is documented as a historical / legacy approach.

**Suggested fix:** Update to recommend `examples` + `[Extensible enum]`
description prefix as the primary method, mentioning `x-extensible-enum` as a
legacy alternative for older OpenAPI versions.

**File:** `json-conventions/SKILL.md`, enum section.

## 5. No dedicated `content-negotiation` skill

**Impact:** Low. The reference docs cover content negotiation in `data-formats.md`
(#244) and `http-headers.md` touches on it briefly (rule 3, Content-Type/Accept).
But there is no skill that walks through designing multiple representations of
the same resource (JSON vs PDF vs HTML for an invoice, language variants via
`Accept-Language`). The `http-headers` skill mentions it in passing but doesn't
cover the design of the representations themselves.

**Suggested fix:** Either (a) add a short `content-negotiation` knowledge skill
covering `Accept`/`Accept-Language`/`Accept-Encoding` with standard media types,
or (b) expand the content-negotiation section in `http-headers` into a
self-contained block.

## 6. `add-resource` lacks cross-reference for collection-item creation

**Impact:** Low. The skill handles singletons, `self` resources, and
sub-resources of a parent item. But when the user wants to `POST` a new child
to a parent *collection* (e.g. create a new address under
`POST /api/v1/customers/{id}/addresses`), the right skill is `add-endpoint`
on that collection path, not `add-resource`. The skill doesn't mention this.

**Suggested fix:** Add a note in Step 4: "If the caller needs to `POST` new
items to a parent collection (not a single resource), use **`add-endpoint`**
on the collection path instead."

**File:** `add-resource/SKILL.md`, Step 4.
