# Fix Plan — 2026-07-15 review findings

Findings from a full review of the reference docs, skills, and packaging.
Organized into four independent work packages (WP1–WP4). Each package can be
handed to a separate agent and executed without the others. Line numbers are
as of commit `2f9b457`; use the quoted text as the authoritative anchor if
lines have drifted. All paths are relative to the repo root
(`plugins/rest-api-design/` abbreviated as `plugin/`).

**Decision baked into this plan:** the `/api` base path is treated as
**required** (MUST). ADAPTATION.md, README.md, and rule #114 already
presuppose it; WP3 promotes rule #135 from SHOULD to MUST to match.

**Global rule for executing agents:** make ONLY the edits listed. Do not
reflow, reformat, or "improve" surrounding text. After each package, grep to
confirm the old text is gone and run the verification step at the end of the
package.

---

## WP1 — Skill citation & contradiction fixes

All files under `plugin/skills/`.

1. **add-collection/SKILL.md:121** and **add-resource/SKILL.md:119** — both
   say `common Money object with a decimal-string amount (#173)`.
   Change `decimal-string` → `decimal-number` in both. (Rule #173 and
   json-conventions define `amount` as a JSON `number` with
   `format: decimal`, not a string. The audit skill was already fixed in
   commit `2f9b457`; these two builders were missed.)

2. **add-collection/SKILL.md:83-89** — the example page object uses relative
   links:
   ```json
   "self":  "/api/v1/orders?cursor=abc",
   "next":  "/api/v1/orders?cursor=def",
   ```
   Pagination skill (L45-46) and rules #161/#217 require **absolute URLs**.
   Change to `https://api.example.com/api/v1/orders?cursor=abc` / `...=def`
   (match the host style used in pagination/SKILL.md's own examples).

3. **json-conventions/SKILL.md:35** — cites `#116/#117/#174` for `etag`.
   **#117 does not exist** in the reference set and #116 (semantic
   versioning) is unrelated. Change the citation to `(#174)` only.

4. **Rule #110 misattributed** — #110 lives in `compatibility.md`, not
   `json-guidelines.md`. Fix the file attribution in:
   - json-conventions/SKILL.md:139
   - pagination/SKILL.md:104
   - add-endpoint/SKILL.md:156
   Move the `#110` mention to a `reference/compatibility.md` line in each
   Reference list (add that line if absent).

5. **Rule #114 misattributed** — #114 (URL versioning) lives in
   `compatibility.md`, not `urls.md`. Fix in:
   - add-endpoint/SKILL.md:152 (also: add-endpoint's Reference list omits
     compatibility.md entirely — add it, covering #110 and #114)
   - http-methods-and-status/SKILL.md:134

6. **new-version/SKILL.md:140-141** — the `reference/compatibility.md` line
   claims `#116 usage`; #116 is in `meta-information.md` (already correctly
   cited at L145). Remove `#116` from the compatibility.md line.

7. **audit/SKILL.md:47-48** — "minor changes do not change the URL (#113)".
   That constraint is stated by **#114**; #113 is "avoid versioning".
   Change `(#113)` → `(#114)`.

8. **add-endpoint/SKILL.md:127-128** — cites `(#133)` for "return the new
   resource URL in the `Location` header". #133 is the generic
   standard-headers rule. Change to `(#180)` (the closest rule covering
   Location semantics).

9. **json-conventions/SKILL.md:58** — cites #112 (`x-extensible-enum`) but
   the skill's Reference list (L139-143) omits `compatibility.md`, where
   #112 lives. Add `reference/compatibility.md — #112 (x-extensible-enum)`
   to the Reference list.

**Verify WP1:** `grep -rn 'decimal-string\|#117\|(#113)' plugin/skills/`
returns nothing; every `#NNN` cited next to a `reference/<file>.md` path in
the six touched skills resolves to an anchor `{#rule-NNN}` or
`<a id="rule-NNN">` in that same file.

---

## WP2 — Reference-doc conversion repairs

All files under `plugin/reference/`. These are AsciiDoc→Markdown conversion
casualties; restore the meaning, matching upstream
(`restful-api-guidelines/chapters/*.adoc`) where noted.

1. **data-formats.md:199** (rule #244) — garbled: "You
   [#172](json-guidelines.md#rule-172) like `application/json` or…".
   The xref text was dropped. Restore: "You
   [should use standard media types (#172)](json-guidelines.md#rule-172)
   like `application/json` or…".

2. **http-headers.md:381** (rule #230, Idempotency-Key) — visible text says
   "See [API Guideline Rule #181][api-230]" but links to #230. Change the
   visible text to `Rule #230`.

3. **http-status-codes-and-errors.md:575** (rule #176) — "`instant` /
   `detail`". RFC 9457 field is `instance` (spelled correctly at L608/L619
   of the same rule). Fix the typo.

4. **events.md:205-214** (rule #197) — two contradictory pasted definitions
   of `ordering_key_fields.items.description`: a JSON Pointer paragraph
   followed by a dot-path paragraph. The example uses dot-path
   (`data.order_change_counter`). Delete the JSON Pointer paragraph; keep
   the dot-path one. (Check upstream `chapters/events.adoc` to confirm which
   is current.)

5. **http-requests.md:562-563** (rule #237) — "The provides the following
   benefits:" → "This provides the following benefits:".

6. **http-requests.md:576** (rule #237) — orphaned fragment: "JSON-specific
   rules and most certainly needs to make use of the `GET`-with-body
   pattern." Restore the lost first half of the sentence from upstream
   `chapters/http-requests.adoc` (rule 237's closing paragraph).

7. **http-headers.md:18** — "you can the standard HTTP" → "you can use the
   standard HTTP".

8. **http-headers.md:28** — `$ref: '#/components/parameters/ETag` — add the
   missing closing single-quote.

9. **events.md:236** — `example: "data.order_number"` is indented one column
   left of its sibling `description` key; fix the indent so the YAML block
   is valid.

10. **compatibility.md:331** (rule #112 YAML example) — the `description:`
    value contains a raw markdown link (`[Extensible enum](https://...)`),
    invalid inside a YAML code block. Replace with plain text:
    `description: Extensible enum — the delivery method...` (keep meaning).

11. **data-formats.md:27-28** — doubled reference "rule [#127](#rule-127)
    ([#127](#rule-127))" appears twice in the table; delete the
    parenthesized duplicate in each.

12. **http-headers.md:88-89** — "([#179](#rule-179) for more details])" —
    remove the stray `]` and add "see": "(see [#179](#rule-179) for more
    details)".

13. **http-status-codes-and-errors.md:38** — "the end endpoint
    specification" → "the endpoint specification".

14. **best-practices.md** — optimistic-locking examples internally
    inconsistent: L173 `GET /orders/BO0000042` vs item id `O0000042`
    (L168/L179) — change `BO0000042` → `O0000042`; L313
    `PUT /block/O0000042` → `PUT /orders/O0000042`.

15. **security.md:104-136 and 147-164** — two HTML comments contain raw
    unconverted AsciiDoc (`<<219, audience>>`, `[source,bnf]`, `|===`).
    Cosmetic (hidden when rendered): either convert the content to markdown
    inside the comments or delete the comments. Prefer deleting unless the
    content adds value beyond the visible text.

**Verify WP2:** the three YAML snippets touched (http-headers.md:28 area,
events.md:236 area, compatibility.md:331 area) parse with
`python3 -c "import yaml,sys; yaml.safe_load(open(f))"` after extracting the
fenced block; `grep -n 'instant`\|the end endpoint\|You \[#172\]' ` over
`plugin/reference/` returns nothing.

---

## WP3 — Adaptation-leak cleanup

Residual upstream-Zalando positions that contradict ADAPTATION.md.

1. **Promote rule #135 to MUST** — `reference/urls.md:6`:
   `## SHOULD use /api as base path {#rule-135}` →
   `## MUST use /api as base path {#rule-135}`. Update the rule body's
   normative verbs to match (SHOULD → MUST where they refer to the base
   path). Then sweep for other statements of #135's strength:
   `grep -rn '135' plugin/ | grep -i should` and fix each (skills cite #135
   in several Reference lists — only change wording that states SHOULD).
   Also update `NOTICE` (both copies, root and plugin/) if they describe
   #135 as SHOULD.

2. **events.md:906-911** (rule #209) — rationale claims REST clients use
   content negotiation for versioned media types, which this adaptation
   forbids (rule #115). Rewrite the rationale to drop the
   versioned-media-type comparison; keep the point that event
   producers/consumers are asynchronous and schema changes are harder to
   coordinate. Add a `> **Adaptation:**` note only if the surrounding
   chapters use one for similar edits.

3. **json-guidelines.md:4-5** — intro: "the 'application/json' media type
   and custom JSON media types defined for APIs." Delete "and custom JSON
   media types defined for APIs" (custom media types are discouraged by
   #172 and forbidden for versioning by #115).

4. **models/request-headers-1.0.0.yaml:84-107** — still ships the Zalando
   proprietary header catalog (`X-Sales-Channel`, `X-Tenant-ID`,
   retailer/CFA prose) contradicting NOTICE item 4's de-branding claim.
   Replace the catalog entries with a single generic example header
   (e.g. `X-Tenant-ID` with a generic multi-tenant description, no
   retailer/CFA/fashion-platform prose), or amend NOTICE item 4 to exclude
   models/. Prefer editing the yaml (NOTICE already promises it).

5. **Retarget upstream links that have local equivalents** (upstream text
   now disagrees with this adaptation, so live links are hazardous):
   - compatibility.md:15 → `deprecation.md`
   - compatibility.md:318 and :331 → `#rule-112`
   - http-requests.md:45 → local `GET with body` section anchor
   - hyper-media.md:148 → `terms.md`
   - performance.md:225 → local `#rule-157`
   Leave upstream links inside YAML header-definition examples
   (deprecation.md / http-headers.md / performance.md models) untouched —
   they intentionally mirror published models.

6. **Dead-narrative pointers** (upstream links dropped in conversion,
   leaving unactionable text):
   - meta-information.md:158 — "(rationale)" dangles; delete the word.
   - meta-information.md:172-173 — "see the API Audience narrative" —
     no such doc here; delete the sentence.
   - meta-information.md:211-214 (rule #223) — mandates a "functional name
     registry" that doesn't exist in this adaptation. Soften to: teams
     SHOULD maintain a registry of functional names to guarantee
     uniqueness; drop the "centralized infrastructure service" claim. Mark
     with an `> **Adaptation:**` note.
   - best-practices.md:117-128 — "A real-world example can be found in an
     internal API repository" plus follow-on bullets analyzing an example
     the reader can't see. Rewrite the bullets as self-contained (describe
     the cursor implementation choice generically) or delete the passage.

7. **Optional, recommended:** add one `/api/v1/...` example to the most
   commonly read chapters (urls.md resource examples, best-practices.md
   `GET /orders`) so sampled examples reinforce the canonical path. Keep
   this to 2-3 edits max; do NOT rewrite every example in the corpus.

**Verify WP3:** `grep -rn 'SHOULD use /api' plugin/` returns nothing;
`grep -rn 'Sales channels are owned\|fashion platform\|CFA' plugin/` returns
nothing; `grep -rn 'opensource.zalando.com' plugin/reference/*.md` hits only
inside fenced YAML example blocks.

---

## WP4 — Skill triggers & portability

1. **add-resource/SKILL.md:3** — description doesn't disambiguate from
   add-collection (users say "add a resource" meaning a collection). Append
   to the description: "…a SINGLE named resource or singleton; if the user
   wants a plural set of items, use add-collection instead."

2. **add-endpoint/SKILL.md:3** — description should state the precondition:
   append "…an operation on a resource path that ALREADY EXISTS in the
   spec; for a new path use add-collection or add-resource." Also state the
   parent-must-exist rule explicitly in the body near Step 2, matching how
   add-collection/add-resource phrase it.

3. **README.md:53** — flagship example `/rest-api-design:add-resource
   orders/line-items` routes to the wrong skill (line-items is a plural
   collection). Change to `/rest-api-design:add-collection
   orders/line-items`.

4. **Portability of `${CLAUDE_PLUGIN_ROOT}`** — every skill's Reference
   section depends on this Claude Code-specific variable, but the repo
   targets agents generally. In each of the 15 SKILL.md files, add one line
   where the variable is first used: "(If `${CLAUDE_PLUGIN_ROOT}` is not
   defined in your environment, the plugin root is the directory two levels
   above this SKILL.md.)" Identical wording in all 15.

5. **Non-portable `$ref`s in generated specs** — these two skills tell the
   agent to embed plugin-local paths into the user's OpenAPI file:
   - error-responses/SKILL.md:31-39 (`$ref: '${CLAUDE_PLUGIN_ROOT}/...problem-1.0.1.yaml#/Problem'`)
   - json-conventions/SKILL.md:110-113 (Money `$ref`)
   Rewrite both examples to match the pattern create-new-api (L99-100) and
   http-headers (L117-121) already use: copy the model into the spec's
   `components/schemas` and `$ref` it locally
   (`#/components/schemas/Problem`, `#/components/schemas/Money`). Keep a
   note that the source of truth is the bundled model file.

6. **audit/SKILL.md:4** — `type: knowledge` while the other named ops are
   `type: builder` and README calls audit a "Named op". Since audit doesn't
   modify artifacts, either introduce `type: named-op` for all four named
   ops or leave audit as knowledge and note the taxonomy in README. Minimal
   fix: leave the value, add a comment line in README's table caption that
   "kind" is descriptive only. (Low priority; pick the minimal fix.)

7. **Cosmetic:** in the 11 skills where the mode-of-operation boilerplate
   line is jammed against the next `##` heading (e.g.
   add-collection/SKILL.md:19-20, json-conventions/SKILL.md:17-18), insert
   a blank line between.

8. **Duplication drift guard (docs-only, no restructuring now):** add a
   short "Maintenance" section to ADAPTATION.md: "The naming/schema/
   status-code rule summaries are duplicated across the builder skills
   (add-collection, add-resource, add-endpoint, create-new-api) and the
   fundamentals skills. When changing any shared rule summary, grep all
   skills for the old wording and update every copy." (A real
   deduplication/shared-fragment refactor is out of scope for this plan.)

**Verify WP4:** `grep -rln 'CLAUDE_PLUGIN_ROOT' plugin/skills/ | wc -l` is
15 and each file contains the fallback sentence;
`grep -rn 'CLAUDE_PLUGIN_ROOT' plugin/skills/error-responses/SKILL.md
plugin/skills/json-conventions/SKILL.md` shows no `$ref:` lines using the
variable; README no longer contains `add-resource orders/line-items`.

---

## Explicitly out of scope

- GitHub-incompatible `{#rule-NNN}` anchors (fine for agent consumption;
  fixing requires a different anchor scheme across ~145 headings).
- `displayName` in plugin.json (harmless extra field).
- Restructuring the duplicated builder-skill rule blocks into a shared
  fragment (guarded by WP4.8 instead).
- changelog.md's original-position text (its header already disclaims it).
