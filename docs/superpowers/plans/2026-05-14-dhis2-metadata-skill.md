# DHIS2 Metadata Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add comprehensive DHIS2 metadata knowledge to the existing `dhis2-app-development` skill — covering the metadata model, querying surface, schema self-discovery, top-7 type recipes, and sharing.

**Architecture:** Extend the existing skill at `skills/dhis2-app-dev/` with a new `references/metadata/` folder (five Markdown files) plus updates to `SKILL.md` routing. Mirrors the existing `ui-patterns.md` + `ui-patterns/` pattern. Documentation only — no app code changes.

**Tech Stack:** Markdown (documentation). Verification uses `WebFetch` against docs.dhis2.org/2.42, Playwright MCP against a public play instance (`play.im.dhis2.org`), and `npx opensrc path dhis2/dhis2-core` for Java controller spot-checks.

**Spec:** `docs/superpowers/specs/2026-05-14-dhis2-metadata-skill-design.md`

---

## Implementer notes (read before starting)

This plan writes documentation, not application code. The TDD-style "write test → run → make pass" cycle is replaced with a documentation rhythm:

1. **Verify** the API claim against a source of truth (docs.dhis2.org, live play instance, or opensrc'd Java source) — **before** writing prose about it.
2. **Draft** the section.
3. **Cross-check** the draft against the verification artifact (every example URL must be one you actually fetched; every property name must be one you actually saw).
4. **Commit** the section.

**Never invent properties, parameters, or response shapes.** If verification doesn't confirm it, leave it out. If you find a discrepancy between docs and live behavior, the live instance wins — call it out as a gotcha in the relevant file.

**Style guide:**
- Match the existing skill's prose style. Read `skills/dhis2-app-dev/references/data-fetching.md` and `skills/dhis2-app-dev/references/ui-patterns.md` first.
- Short, direct sentences. No filler. No emojis.
- Code examples cap at ~25 lines; longer ones get split into named subsections.
- Every example uses TypeScript with the `useApiDataQuery` / `useDataEngine` patterns already established in `data-fetching.md`.
- Use `displayName` (not `name`) in every example involving user-visible labels.

**Verification toolset:**
- `WebFetch` for docs.dhis2.org pages. Target the **2.42** docs explicitly — URLs typically follow the form `https://docs.dhis2.org/en/develop/using-the-api/dhis-core-version-242/<page>.html`. Confirm the exact base in Task 1.
- Playwright MCP (`mcp__plugin_playwright_playwright__browser_navigate` + `browser_evaluate`) against the public play instance for verifying live API responses. The current stable play instance URL is identified in Task 1 — typically `https://play.im.dhis2.org/stable-2-42/` with login `admin` / `district`.
- `npx opensrc path dhis2/dhis2-core@2.42` for the cached Java source.

---

## Task 1: Verification pass — establish ground truth

**Files:** None created in this task. Output is a verification notes file used by later tasks.

- Create: `docs/superpowers/plans/2026-05-14-dhis2-metadata-skill.verification.md` (working notes, **not committed** — added to `.gitignore` if needed)

- [ ] **Step 1: Confirm docs base URL for 2.42**

Run via `WebFetch`:
- `https://docs.dhis2.org/en/develop/using-the-api/dhis-core-version-242/metadata.html` — confirm reachable
- `https://docs.dhis2.org/en/develop/using-the-api/dhis-core-version-242/metadata-object-filter.html` — confirm reachable
- `https://docs.dhis2.org/en/develop/using-the-api/dhis-core-version-242/metadata-gist.html` — confirm reachable
- `https://docs.dhis2.org/en/develop/using-the-api/dhis-core-version-242/sharing.html` — confirm reachable
- `https://docs.dhis2.org/en/develop/using-the-api/dhis-core-version-242/schemas.html` — confirm reachable

If any 404s, search docs.dhis2.org for the equivalent page and record the actual path. Persist the resolved URLs in the verification notes file.

- [ ] **Step 2: Identify and reach the live play instance**

Try `https://play.im.dhis2.org/stable-2-42/api/me` (no auth) — expect a 401. Then via Playwright, navigate to `https://play.im.dhis2.org/stable-2-42/dhis-web-commons/security/login.action`, log in as `admin` / `district`, then navigate to `/api/system/info.json` and confirm `version` starts with `2.42`.

If the URL is wrong, search `play.im.dhis2.org` for the correct stable-2-42 path. Record the resolved base URL in the verification notes.

- [ ] **Step 3: Cache the dhis2-core source**

Run: `npx opensrc path dhis2/dhis2-core@2.42`
Expected: prints an absolute path under `~/.opensrc/repos/github.com/dhis2/dhis2-core/`. Record the path in the verification notes.

- [ ] **Step 4: Snapshot `/api/schemas` from the live instance**

Via Playwright, navigate (authenticated) to `<PLAY_BASE>/api/schemas.json?fields=name,plural,singular,klass,shareable,relativeApiEndpoint,persisted` and save the JSON response into the verification notes. This becomes the reference list of valid metadata types and singular/plural names.

- [ ] **Step 5: Snapshot one example schema in full**

Via Playwright, navigate to `<PLAY_BASE>/api/schemas/dataElement.json` and save the full response. Used as the reference example in `schemas.md`.

- [ ] **Step 6: Verify the field-filter operator list**

Read the docs page from step 1 and the controller source. Specifically open:
- `<CORE>/dhis-2/dhis-api/src/main/java/org/hisp/dhis/query/operators/` — directory listing of operator classes
- `<CORE>/dhis-2/dhis-api/src/main/java/org/hisp/dhis/schema/descriptors/` — for property type info

Cross-check against the docs page and record the canonical operator list (and any docs-vs-source discrepancies).

- [ ] **Step 7: Verify the sharing access-string format**

Read `<CORE>/dhis-2/dhis-api/src/main/java/org/hisp/dhis/security/acl/AccessStringHelper.java` (or equivalent — find via `rg "AccessStringHelper" "$CORE/dhis-2"`). Record:
- Exact character count
- Position-to-permission mapping
- Whether positions 5-8 are reserved/unused, or have semantics

Also fetch sharing for one object from the live instance: `<PLAY_BASE>/api/sharing?type=dataElement&id=<some-uid>` (pick any UID from the dataElements list endpoint). Confirm the response shape.

- [ ] **Step 8: Verify Gist API behavior**

Hit `<PLAY_BASE>/api/dataElements/gist.json?pageSize=2&fields=id,displayName,categoryCombo[id,displayName]` against the live instance. Save response. Note differences from the regular endpoint (envelope shape, default fields, supported filter operators).

- [ ] **Step 9: Verify `/api/identifiableObjects/{uid}` exists and its response shape**

Take a UID from the dataElements list, then hit `<PLAY_BASE>/api/identifiableObjects/<uid>.json` and save the response. Note the type-resolution behavior.

- [ ] **Step 10: Commit only the notes that should persist**

The verification notes file (`docs/superpowers/plans/2026-05-14-dhis2-metadata-skill.verification.md`) stays uncommitted — it's a scratchpad. Subsequent tasks reference it; the final committed artifacts are the metadata reference files themselves.

No commit at the end of this task.

---

## Task 2: Write `references/metadata.md` (entry)

**Files:**
- Create: `skills/dhis2-app-dev/references/metadata.md`

- [ ] **Step 1: Verify identifier facts**

Confirm against the live instance and source:
- UID regex (typically `^[A-Za-z][A-Za-z0-9]{10}$`) — find in `<CORE>/dhis-2/dhis-api/src/main/java/org/hisp/dhis/common/CodeGenerator.java`
- `displayName` resolution behavior — find the `Translation` / `displayName` logic. Confirm: `displayName` is server-resolved against the active user locale; `name` is the source string.

Note discrepancies in the verification notes.

- [ ] **Step 2: Draft `references/metadata.md` with these sections**

Section headings (in order):

1. `# DHIS2 Metadata`
2. `## Metadata vs data` — 1 short paragraph, link back to `data-fetching.md` for caching strategy
3. `## The metadata domain map` — one paragraph + a Mermaid graph (or ASCII tree if Mermaid is not rendered elsewhere in the skill — check what `ui-patterns.md` uses) showing relationships between Org Units, Data Elements, Indicators, Programs (event vs tracker, with TET/TEA/Program Stages), Datasets, Option Sets, Category Combos
4. `## Identifiers` — UID format (with regex), `code`, `name` vs `shortName` vs `displayName`, translations, `href`
5. `## /api/metadata vs per-type endpoints` — one paragraph each; pointer to deferred bulk-import section at the bottom
6. `## When to read which sub-doc` — routing table to the four sub-files, mirroring SKILL.md style
7. `## Out of scope (deferred)` — one paragraph naming bulk import / dependency export / metadata versioning and pointing to where they'd be added later

Style: ~150 lines total. Sentences not paragraphs. Tables where data is tabular.

Required content in the Identifiers section:

```markdown
DHIS2 metadata objects are addressed by **UID** — an opaque 11-character identifier
matching `/^[A-Za-z][A-Za-z0-9]{10}$/`. Generated server-side; treat as opaque.

Each object also carries:
- `code` — optional, user-defined, intended to be human-readable and stable across
  instances. Use for cross-instance references when UIDs differ.
- `name` — the source name in the object's authoring locale. Required on most types.
- `shortName` — abbreviated label for narrow contexts (tables, charts).
- `displayName` — `name` resolved through the user's active translation. **Always
  prefer `displayName` in UIs** — `name` shows the wrong language to non-default users.
- `displayShortName` — same, for `shortName`.
- `translations` — array of `{ locale, property, value }` triples driving `displayName` /
  `displayShortName`.
```

Required content in the When-to-read table:

```markdown
| Task | Read |
|------|------|
| Build a query, filter, or list view | `metadata/querying.md` |
| Discover the shape of a type at runtime | `metadata/schemas.md` |
| Create or edit a specific metadata type | `metadata/common-types.md` |
| Set or change sharing | `metadata/sharing.md` |
```

- [ ] **Step 3: Cross-check the draft against the verification notes**

For each fact in the draft (UID regex, displayName resolution, list of types in the domain map), confirm it matches what you saw in source/live. Fix any drift.

- [ ] **Step 4: Commit**

```bash
git add skills/dhis2-app-dev/references/metadata.md
git commit -m "docs: add metadata.md entry with model overview and routing

Covers metadata vs data, the domain map, identifiers (UID, code, name
variants, translations), per-type vs bulk endpoints, and a routing
table to the sub-files."
```

---

## Task 3: Write `references/metadata/querying.md`

**Files:**
- Create: `skills/dhis2-app-dev/references/metadata/querying.md`

This is the deepest file. The draft must be backed by verified examples — every URL shown should be one you actually executed against the play instance.

- [ ] **Step 1: Run verification queries against the live play instance**

Execute (via Playwright) and save responses:

1. `<PLAY_BASE>/api/dataElements.json?fields=id,displayName&pageSize=2`
2. `<PLAY_BASE>/api/dataElements.json?fields=id,displayName,categoryCombo[id,displayName,categories[id,displayName]]&pageSize=2`
3. `<PLAY_BASE>/api/dataElements.json?fields=:identifiable&pageSize=2`
4. `<PLAY_BASE>/api/dataElements.json?fields=displayName~rename(label),id&pageSize=2`
5. `<PLAY_BASE>/api/dataElements.json?fields=id,displayName,dataSetElements::size&pageSize=2`
6. `<PLAY_BASE>/api/dataElements.json?filter=displayName:ilike:vaccine&fields=id,displayName&pageSize=5`
7. `<PLAY_BASE>/api/dataElements.json?filter=valueType:in:[NUMBER,INTEGER]&filter=domainType:eq:AGGREGATE&fields=id,displayName,valueType,domainType&pageSize=5`
8. `<PLAY_BASE>/api/dataElements.json?filter=displayName:ilike:bcg&filter=valueType:eq:NUMBER&rootJunction=OR&fields=id,displayName,valueType&pageSize=5`
9. `<PLAY_BASE>/api/programs.json?filter=programStages.programStageDataElements.dataElement.valueType:eq:NUMBER&fields=id,displayName&pageSize=3` (nested filter)
10. `<PLAY_BASE>/api/dataElements.json?order=displayName:asc&fields=id,displayName&pageSize=3`
11. `<PLAY_BASE>/api/dataElements.json?paging=false&fields=id` (note response size — call this out in the doc)
12. `<PLAY_BASE>/api/dataElements/gist.json?fields=id,displayName,categoryCombo[id,displayName]&pageSize=3`
13. `<PLAY_BASE>/api/metadata.json?dataElements=true&programs=true&fields=:owner` (cross-type — note response shape)
14. `<PLAY_BASE>/api/identifiableObjects/<uid>.json` (pick a known UID from step 1)

For each: confirm the URL works and save the response. If any return an error, investigate before writing the doc.

- [ ] **Step 2: Compile the canonical operator list**

Cross-reference docs and source (from Task 1 step 6). Produce the final operator table. Likely includes: `eq`, `!eq`, `ieq`, `like`, `!like`, `ilike`, `!ilike`, `$like`, `like$`, `in`, `!in`, `gt`, `ge`, `lt`, `le`, `null`, `!null`, `empty`, `!empty`, `token`, `!token`. Confirm exactly what's available in 2.42 — operator lists change. Adjust based on what source shows.

- [ ] **Step 3: Draft `querying.md` with these sections**

Section headings (in order):

1. `# Querying DHIS2 Metadata`
2. `## Field selection (\`fields\`)`
   - When to use `fields` (always)
   - Nested fields with brackets
   - Presets table: `:all`, `:owner`, `:identifiable`, `:nameable`, `:simple` — what each returns
   - Field transforms: `~rename(alias)`, `::size`, `::isNotEmpty`, `~paging(page,pageSize)`, `!field` exclusion
   - Worked example: query 2 from step 1 (nested) + query 4 (rename) shown side-by-side with output
3. `## Filtering (\`filter\`)`
   - Format: `property:operator:value`
   - Full operator table (from step 2)
   - AND vs OR (`rootJunction=OR`)
   - Nested filters across references (query 9)
   - Worked examples: queries 6, 7, 8, 9 from step 1
   - Practical patterns subsection: search-as-you-type (with dynamic filter array snippet, mirroring `data-fetching.md`'s pattern); filtering by sharing/access
4. `## Ordering, paging, total counts`
   - `order` syntax + secondary order
   - `page`, `pageSize`, `paging=false` (with the caveat that some endpoints cap or refuse this — verify against the data you got in step 1 query 11)
   - `totalPages=true` and its cost
5. `## The Gist API`
   - What `/api/<type>/gist` returns (compact response, default fields differ)
   - Reference navigation: `/api/users/<uid>/userGroups/gist`
   - What's not supported in gist (lighter filter/fields surface — list specifics from your verification of query 12)
6. `## Cross-type queries`
   - `/api/metadata?dataElements=true&programs=true&fields=...` for snapshots — note the response shape from query 13
   - `/api/identifiableObjects/{uid}` for resolving an unknown UID's type — example from query 14

Style: ~400 lines total. Every example is a real URL with a real response excerpt (kept short).

Required content in the operators table format:

```markdown
| Operator | Meaning | Example |
|----------|---------|---------|
| `eq` | Equals | `filter=valueType:eq:NUMBER` |
| `!eq` | Not equals | `filter=valueType:!eq:NUMBER` |
| `ieq` | Case-insensitive equals | `filter=code:ieq:abc` |
| `like` | Substring match, case-sensitive | `filter=name:like:Vacc` |
| `ilike` | Substring match, case-insensitive | `filter=displayName:ilike:vaccine` |
| ... | ... | ... |
```

(Complete the table from your step 2 work.)

Required content for the search-as-you-type pattern:

```typescript
const filters = [
    ...(searchTerm ? [`displayName:ilike:${searchTerm}`] : []),
    'valueType:in:[NUMBER,INTEGER]',
    'domainType:eq:AGGREGATE',
];

const { data } = useApiDataQuery<{ dataElements: DataElement[] }>({
    queryKey: ['dataElements', 'list', searchTerm],
    query: {
        resource: 'dataElements',
        params: {
            fields: 'id,displayName,valueType',
            filter: filters,
            order: 'displayName:asc',
            page: 1,
            pageSize: 50,
        },
    },
    cacheTime: Infinity,
    staleTime: Infinity,
});
```

- [ ] **Step 4: Cross-check every URL example against verification artifacts**

For each example in the file, confirm the URL is one you actually fetched in step 1. If you reference a parameter you didn't verify, either verify it now or remove it.

- [ ] **Step 5: Commit**

```bash
git add skills/dhis2-app-dev/references/metadata/querying.md
git commit -m "docs: add metadata querying reference

Covers fields/filter/order/paging in depth, the Gist API, and
cross-type queries. Every example URL is verified against a live
DHIS2 2.42 instance."
```

---

## Task 4: Write `references/metadata/schemas.md`

**Files:**
- Create: `skills/dhis2-app-dev/references/metadata/schemas.md`

- [ ] **Step 1: Run targeted verification queries**

Already have `/api/schemas.json` summary and `/api/schemas/dataElement.json` full response from Task 1 steps 4-5. Additionally fetch:

1. `<PLAY_BASE>/api/schemas/dataElement/properties.json` — confirm whether per-property listing exists at this path or another
2. `<PLAY_BASE>/api/schemas/dataElement/properties/categoryCombo.json` — confirm shape (or find the correct path if 404)

If either path 404s, search the source under `<CORE>/dhis-2/dhis-web-api/src/main/java/org/hisp/dhis/webapi/controller/schema/` for the schema controller and find the correct routes. Record findings in the verification notes.

- [ ] **Step 2: Identify the key properties of a schema descriptor**

From `/api/schemas/dataElement.json`, list the top-level fields and what they mean:
- `klass`, `name`, `plural`, `singular`, `shareable`, `relativeApiEndpoint`, `persisted`, `embeddedObject`, `dataShareable`
- `properties[]` with each property's `name`, `fieldName`, `propertyType`, `klass`, `required`, `unique`, `writable`, `readable`, `persisted`
- For reference properties: `propertyType: REFERENCE` plus `klass` pointing to the target type
- For collection properties: `propertyType: COLLECTION` plus `itemPropertyType`, `itemKlass`

Confirm each field appears in your snapshot. Note any 2.42-specific additions.

- [ ] **Step 3: Draft `schemas.md` with these sections**

Section headings (in order):

1. `# Self-discovering metadata types with /api/schemas`
2. `## What /api/schemas returns` — list of all metadata types with key descriptor fields; brief description of each top-level descriptor field (from step 2)
3. `## Per-type and per-property endpoints` — `/api/schemas/<type>` and (confirmed in step 1) the per-property route
4. `## The AI workflow` — when asked about an unfamiliar type, this is the first endpoint to hit. Recipe:
   - GET `/api/schemas/<type>`
   - Read `properties[]` to find required fields (`required: true`), references (`propertyType: REFERENCE`), collections, embedded objects
   - Use this to compose queries (which `fields` are valid) and payloads (which fields are writable + required)
5. `## When schemas isn't enough` — schemas tell you the shape, not the validation rules or cross-property semantics. For those, fall back to the controller via opensrc. Link to `data-fetching.md` step 1.
6. `## Worked example: discover the writable surface of dataElement`

Required content for the worked example:

```markdown
### Worked example: discover the writable surface of `dataElement`

```bash
curl '<PLAY_BASE>/api/schemas/dataElement.json?fields=properties[name,fieldName,propertyType,klass,required,writable,persisted,itemKlass]'
```

Returns (excerpt):

```json
{
    "properties": [
        { "name": "name", "fieldName": "name", "propertyType": "TEXT", "required": true, "writable": true, "persisted": true },
        { "name": "valueType", "fieldName": "valueType", "propertyType": "CONSTANT", "required": true, "writable": true, "persisted": true },
        { "name": "categoryCombo", "fieldName": "categoryCombo", "propertyType": "REFERENCE", "klass": "org.hisp.dhis.category.CategoryCombo", "required": true, "writable": true, "persisted": true }
    ]
}
```

Reading this tells the AI: a valid `dataElement` create payload **must** include
`name`, `valueType`, and a `categoryCombo` reference, plus whatever non-required
fields the app cares to set.
```

(Fill in the actual response excerpt from your verification snapshot.)

Style: ~120 lines total.

- [ ] **Step 4: Cross-check schema field names against the live snapshot**

Open the saved `/api/schemas/dataElement.json` and verify every property/field name shown in the doc actually appears in the response.

- [ ] **Step 5: Commit**

```bash
git add skills/dhis2-app-dev/references/metadata/schemas.md
git commit -m "docs: add /api/schemas self-discovery reference

Teaches the AI to hit /api/schemas/<type> before composing queries
or payloads. Worked example uses real response data from a 2.42
play instance."
```

---

## Task 5: Write `references/metadata/common-types.md`

**Files:**
- Create: `skills/dhis2-app-dev/references/metadata/common-types.md`

Recipes for the top 7 metadata types. Each recipe is ~15-25 lines.

- [ ] **Step 1: Per-type verification — fetch one example of each type from the live instance**

For each of the 7 types, fetch a sample with `:owner` fields and save:

1. `<PLAY_BASE>/api/organisationUnits.json?fields=:owner&pageSize=1`
2. `<PLAY_BASE>/api/dataElements.json?fields=:owner&pageSize=1`
3. `<PLAY_BASE>/api/indicators.json?fields=:owner&pageSize=1`
4. `<PLAY_BASE>/api/programs.json?fields=:owner&pageSize=1`
5. `<PLAY_BASE>/api/trackedEntityTypes.json?fields=:owner,trackedEntityTypeAttributes[*]&pageSize=1`
6. `<PLAY_BASE>/api/optionSets.json?fields=:owner,options[id,code,name,sortOrder]&pageSize=1`
7. `<PLAY_BASE>/api/categoryCombos.json?fields=:owner,categories[id,displayName],categoryOptionCombos[id,displayName]&pageSize=1`

For each, identify which fields are required (cross-reference with `/api/schemas/<singular>.json`).

- [ ] **Step 2: Verify each minimal-create payload by hitting the schema**

For each type, the minimal create payload includes only required + commonly-needed fields. Use the schema's `required: true` properties as the floor. Note any fields where `required` depends on context (e.g., `categoryCombo` is required on data elements but has a sensible default UID).

Identify the "default" CategoryCombo UID via `<PLAY_BASE>/api/categoryCombos.json?filter=name:eq:default&fields=id`. Record it for use in examples.

- [ ] **Step 3: Draft `common-types.md` with one section per type**

Section headings (in order):

1. `# Common metadata types — recipes`
2. `## Organisation Units`
3. `## Data Elements`
4. `## Indicators`
5. `## Programs (event and tracker)`
6. `## Tracked Entity Types & Attributes`
7. `## Option Sets & Options`
8. `## Category Combos, Categories, Category Options`

Each section follows the same micro-structure:

```markdown
## [Type Name]

**Endpoint:** `/api/<plural>` (singular: `<singular>`)

**Key fields:** [comma-separated list with one-line semantics each, only the
fields apps usually touch]

**Required references:** [what other metadata must exist before this can be
created; cite the schema]

**Common gotchas:**
- [Gotcha 1]
- [Gotcha 2]

**Minimal create payload:**

```typescript
const payload = {
    // verified-minimum shape — copy/adapt
};

const createMutation = {
    resource: '<plural>',
    type: 'create' as const,
    data: payload,
};
```
```

Required content per type (the gotchas — these are the things AI gets wrong):

- **Organisation Units:** Parent must exist before creation. `path` and `level` are derived server-side from `parent` — never set them in a create payload. Geometry uses GeoJSON.
- **Data Elements:** AGGREGATE elements participate in datasets; TRACKER elements participate in program stages — different validation rules apply (notably around `aggregationType` and `categoryCombo`). The "default" category combo UID is `<insert verified UID>` — use it when the data element has no disaggregation.
- **Indicators:** `numerator` / `denominator` use the expression syntax `#{dataElementUid}` for an aggregate DE total and `#{dataElementUid.categoryOptionComboUid}` for a specific disaggregation. The server validates that referenced UIDs exist. Indicator values are computed at query time — they're not stored.
- **Programs:** `programType=WITH_REGISTRATION` is tracker (needs `trackedEntityType`); `WITHOUT_REGISTRATION` is event (no TET; one stage). `programStages` is owned by the program; `programTrackedEntityAttributes` link existing TEAs to this program (the TEAs themselves live on the TET).
- **Tracked Entity Types & Attributes:** TEAs are global, owned by no single TET (despite the name); `trackedEntityTypeAttributes` on a TET is an association object linking the TET to TEAs. Per-program uniqueness is set on the `programTrackedEntityAttribute`, not the TEA itself.
- **Option Sets & Options:** Options are owned by the set (deleting the set deletes its options). `code` is the persisted value, used in data values — case-sensitive. `sortOrder` is 1-indexed and controls display order.
- **Category Combos:** Combo → Categories → CategoryOptions is the triple. `categoryOptionCombos` (note the plural-with-extra-Os) are the materialized cross-product, auto-generated server-side — don't try to set them in a create. Changing `categoryCombo` on a data element that already has data values is destructive — the server will refuse without `force=true`.

Style: ~300 lines total. Each section short and recipe-like.

- [ ] **Step 4: Cross-check payload shapes against live snapshots**

For each `Minimal create payload`, open the saved live response from step 1 and confirm every field name in the payload appears as a writable property in `/api/schemas/<type>.json`. Remove any invented fields.

- [ ] **Step 5: Commit**

```bash
git add skills/dhis2-app-dev/references/metadata/common-types.md
git commit -m "docs: add recipes for the top 7 metadata types

Org units, data elements, indicators, programs, tracked entity
types/attributes, option sets, and category combos. Each recipe
covers endpoint, key fields, required references, common gotchas,
and a minimal verified create payload."
```

---

## Task 6: Write `references/metadata/sharing.md`

**Files:**
- Create: `skills/dhis2-app-dev/references/metadata/sharing.md`

- [ ] **Step 1: Run targeted verification queries**

Pick a data element UID from earlier verification. Then:

1. `<PLAY_BASE>/api/sharing?type=dataElement&id=<UID>` — save the GET response; note the exact shape (`object` envelope vs flat).
2. Confirm the access-string format using the source notes from Task 1 step 7. If unclear, also do: `<PLAY_BASE>/api/sharing?type=visualization&id=<some-viz-uid>` (visualizations are dataShareable; data elements may or may not be — confirms which positions of the string mean what).
3. Try POST: change one of the test object's `publicAccess` from current value to `r-------` and back. Confirm the format the API accepts.

Save responses and exact request bodies.

- [ ] **Step 2: Verify which types are shareable**

From the `/api/schemas.json` snapshot, list all types where `shareable: true` and where `dataShareable: true`. Note the difference (`dataShareable` means the access string's data positions matter; otherwise they're ignored).

- [ ] **Step 3: Draft `sharing.md` with these sections**

Section headings (in order):

1. `# Sharing metadata`
2. `## The sharing model` — every shareable object has `publicAccess`, `userAccesses[]`, `userGroupAccesses[]`, `externalAccess`, owner (`createdBy` / `user`)
3. `## The access string` — 8-char format per Task 1 step 7 findings; position table; cheat sheet of common combinations
4. `## Reading sharing — GET /api/sharing` — uses singular schema name in `type`
5. `## Writing sharing — POST /api/sharing` — same shape; the server accepts the response body of GET back as input
6. `## Which types are shareable` — pointer to `shareable` and `dataShareable` flags on `/api/schemas` (don't reproduce the list — let schemas be the source of truth)
7. `## Common patterns` — code snippets for: make public read-only; share with a user group RW; remove all user/group accesses; copy sharing from another object

Required content for the access string section (fill in with verified findings):

```markdown
## The access string

DHIS2 represents access as an 8-character string. Each pair of characters is
either `r-`, `rw`, or `--`:

| Positions | Meaning | Notes |
|-----------|---------|-------|
| 1-2 | Metadata read/write | Applies to all shareable types |
| 3-4 | Data read/write | Only meaningful when the type has `dataShareable: true` (e.g. data elements, visualizations) |
| 5-6 | <verified meaning or "Reserved"> | |
| 7-8 | <verified meaning or "Reserved"> | |

Common combinations:

| String | Meaning |
|--------|---------|
| `rwrw----` | Full metadata + data access |
| `r-r-----` | Read metadata + read data |
| `rw------` | Edit metadata; no data access |
| `r-------` | Read metadata only |
| `--------` | No access |
```

(Fill in the position semantics from your Task 1 step 7 findings.)

Required content for the patterns section:

```typescript
// Share a metadata object publicly read-only
const sharing = {
    object: {
        publicAccess: 'r-r-----',
        externalAccess: false,
        userAccesses: [],
        userGroupAccesses: [],
    },
};

const updateSharing = {
    resource: 'sharing',
    type: 'update' as const,
    params: { type: 'dataElement', id },
    data: sharing,
};
```

Note: the existing `data-fetching.md` uses `useDataEngine` for mutations — follow that pattern. Sharing-write is a POST under the hood but uses `type: 'update'` in app-runtime's mutation shape because the engine looks at the HTTP method via the resource definition.

Verify this against the source if uncertain: look in `<CORE>/dhis-2/dhis-web-api/src/main/java/org/hisp/dhis/webapi/controller/sharing/` for the sharing controller and confirm the HTTP method.

Style: ~150 lines.

- [ ] **Step 4: Cross-check the request/response examples against the saved live request/response**

Open the saved request/response from step 1 step 3. Confirm the example body in the doc matches what the server actually accepts.

- [ ] **Step 5: Commit**

```bash
git add skills/dhis2-app-dev/references/metadata/sharing.md
git commit -m "docs: add metadata sharing reference

Covers the sharing model, the 8-char access string format, GET/POST
/api/sharing, what's shareable, and common patterns (public read-only,
share-with-group, remove-all, copy-from)."
```

---

## Task 7: Update SKILL.md

**Files:**
- Modify: `skills/dhis2-app-dev/SKILL.md`

- [ ] **Step 1: Read the current SKILL.md**

Read the whole file to confirm exact wording of nearby sections.

- [ ] **Step 2: Add new rows to the scenario table**

Locate the existing table (currently ends with the "Handle API differences across DHIS2 versions" row). Add these rows immediately after:

```markdown
| Understand the DHIS2 metadata model (types, identifiers, sharing) | `references/metadata.md` |
| Search, list, or filter metadata (any type) | `references/metadata.md` → `references/metadata/querying.md` |
| Discover the shape of a metadata type at runtime | `references/metadata/schemas.md` |
| Build a UI to create/edit a specific metadata type | `references/data-fetching.md` → `references/metadata/common-types.md` |
| Set or change sharing on metadata | `references/metadata/sharing.md` |
```

- [ ] **Step 3: Add a new rule**

In the Rules section, after the existing `Use displayName instead of name…` bullet, add:

```markdown
- **Metadata correctness.** Treat UIDs as opaque 11-char IDs (`/^[A-Za-z][A-Za-z0-9]{10}$/`). Never invent metadata properties or query parameters — verify with `/api/schemas/<type>` (see `references/metadata/schemas.md`) or the controller source via opensrc before composing a query or payload.
```

- [ ] **Step 4: Commit**

```bash
git add skills/dhis2-app-dev/SKILL.md
git commit -m "docs: route metadata scenarios to new references

Adds five scenario rows pointing at the new metadata/ folder and a
new rule requiring schema/source verification before composing
metadata queries or payloads."
```

---

## Task 8: Cross-link audit

**Files:** Read-only verification of all files modified or created in tasks 2-7.

- [ ] **Step 1: List every cross-reference**

Run: `rg -n '(references/|metadata/|data-fetching\.md|ui-patterns)' skills/dhis2-app-dev/SKILL.md skills/dhis2-app-dev/references/metadata.md skills/dhis2-app-dev/references/metadata/`

Expected: every reference resolves to a real file. Specifically check:
- `references/metadata.md` exists
- `references/metadata/querying.md`, `schemas.md`, `common-types.md`, `sharing.md` all exist
- Cross-links from `metadata.md` to the sub-files use the correct relative path (`metadata/querying.md`, not `references/metadata/querying.md` — depends on the file's perspective)
- Cross-links from sub-files back to `data-fetching.md` use `../data-fetching.md`

- [ ] **Step 2: Fix any broken links**

If any reference is wrong, fix it in the relevant file. Commit the fix as a separate commit:

```bash
git add <fixed files>
git commit -m "docs: fix cross-reference paths in metadata reference"
```

- [ ] **Step 3: Sanity-read each file end-to-end**

Open each file and read top to bottom. Look for:
- Sentences that don't parse
- Examples that reference variables not defined elsewhere in the same file
- Tables with missing rows or columns
- Code blocks with unclosed backticks
- Any "TBD" / "TODO" / "verify later" that slipped through

Fix issues inline. Commit:

```bash
git add <files with fixes>
git commit -m "docs: prose and example fixes in metadata reference"
```

- [ ] **Step 4: Final verification — render the skill mentally**

Pretend you're an AI assistant given a fresh task: "Build a page that lets a user search and edit data elements." Walk through which files you'd read in order, and confirm:
1. SKILL.md routes you correctly
2. `data-fetching.md` gives you the hook pattern
3. `metadata/common-types.md` gives you the data-element specifics
4. `metadata/querying.md` gives you the filter syntax for search
5. `metadata/schemas.md` is there if you need to discover anything else
6. `metadata/sharing.md` is there if the page needs a sharing UI

If any of these aren't covered, file a follow-up task. Otherwise, the work is complete.

---

## Self-review checklist (for the plan author, before handing off)

- [ ] Every task has exact file paths.
- [ ] Every step that writes content shows the structure (headings, key tables, required snippets) — no "write the section about X" without saying what's in it.
- [ ] Verification artifacts (URLs, source paths) are listed concretely per task.
- [ ] Every cross-reference between tasks names a real file or section.
- [ ] No "TBD" / "fill in later" / "similar to Task N".
- [ ] Commit messages are pre-written.
- [ ] Style guide is named (`data-fetching.md` and `ui-patterns.md` as exemplars).
- [ ] Scope matches the spec — five files + SKILL.md update, no scope creep into bulk import.
