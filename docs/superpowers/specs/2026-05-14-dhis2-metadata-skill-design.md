# DHIS2 Metadata Knowledge — Skill Design

**Date:** 2026-05-14
**Scope:** Extend the existing `dhis2-app-development` skill with comprehensive DHIS2 metadata knowledge so an AI assistant can confidently search, read, create, and change metadata in DHIS2 apps.
**Out of scope (deferred):** Bulk import / migration workflows (`/api/metadata` POST, dependency export, metadata versioning) — pointer only.

## Goals

1. Give an AI assistant complete, accurate working knowledge of the DHIS2 metadata model: types, identifiers, references, sharing, translations.
2. Cover the **query surface** in full — `fields`, `filter`, `order`, paging, Gist API, cross-type lookup — with verified examples.
3. Teach the AI to **self-discover** unfamiliar types via `/api/schemas` instead of guessing.
4. Provide concise, recipe-style coverage of the **top 7 metadata types** used in real apps.
5. Cover **sharing** — the access-string format, the `/api/sharing` endpoint, and common patterns.

A future user of the skill should be able to ask "build a UI to edit data elements" or "find all indicators referencing this data element" and get correct code on the first attempt.

## Non-goals

- A per-type reference manual covering every metadata type. The long tail is handled by `/api/schemas`.
- Coverage of bulk import, migration, or metadata sync. A pointer to where this would live, no recipes.
- Replacing the existing `data-fetching.md` content. The metadata files build on it.

## Target version

DHIS2 **2.42** (current stable). Version-specific notes use the existing `useFeature` / `FEATURES` pattern from `data-fetching.md`. Where the API changed (e.g., access-string format in 2.36, gist endpoint additions), the change is called out inline.

## Structure

New `references/metadata/` folder inside the existing skill, mirroring the `ui-patterns.md` + `ui-patterns/` pattern already used:

```
skills/dhis2-app-dev/
├── SKILL.md                                       (updated — new routing rows + rule)
└── references/
    ├── metadata.md                                (NEW — entry: model + routing)
    └── metadata/
        ├── querying.md                            (NEW — fields, filter, gist, cross-type)
        ├── schemas.md                             (NEW — /api/schemas self-discovery)
        ├── common-types.md                        (NEW — recipes for top 7 types)
        └── sharing.md                             (NEW — access strings + /api/sharing)
```

## SKILL.md changes

Add rows to the scenario table:

| Scenario | References (read in order) |
|----------|---------------------------|
| Understand the DHIS2 metadata model (types, identifiers, sharing) | `references/metadata.md` |
| Search, list, or filter metadata (any type) | `references/metadata.md` → `references/metadata/querying.md` |
| Discover the shape of a metadata type at runtime | `references/metadata/schemas.md` |
| Build a UI to create/edit a specific metadata type | `references/data-fetching.md` → `references/metadata/common-types.md` |
| Set or change sharing on metadata | `references/metadata/sharing.md` |

Add a new rule to the Rules section:

> **Metadata correctness.** Use `displayName` (not `name`) in UIs. Treat UIDs as opaque 11-char IDs (`/^[A-Za-z][A-Za-z0-9]{10}$/`). Never invent metadata properties — verify with `/api/schemas/<type>` or the controller source via opensrc before composing a query or payload.

## File-by-file content

### `references/metadata.md` (entry)

Short. Sections:

- **Metadata vs data.** Recap and link back to `data-fetching.md` caching strategy.
- **The domain map.** One paragraph + simple ASCII/Mermaid relationship sketch covering: org units (hierarchical), data elements (used in datasets or program stages), indicators (expressions over data elements), programs (event vs tracker, with TETs/TEAs), datasets, option sets, category combos.
- **Identifiers.** UID (11-char, regex above), `code` (user-defined), `name` / `shortName` / `displayName` (localized via translations), `href`.
- **`/api/metadata` vs per-type endpoints.** Briefly: per-type (`/api/dataElements`, `/api/programs`) is the usual route; `/api/metadata` is bulk read/write. Pointer to deferred section.
- **Routing table.** When to read which sub-file.
- **Out-of-scope pointer.** Bulk import / dependency export / versioning live here if added later.

Length target: ~150 lines.

### `references/metadata/querying.md`

The deepest file. Sections:

- **Field selection (`fields`).** Always specify; nested fields; presets (`:all`, `:owner`, `:identifiable`, `:nameable`, `:simple`); field transforms (`~rename`, `::size`, `::isNotEmpty`, `~paging(N,M)`); exclusion with `!field`. Worked example with a nested query.
- **Filtering (`filter`).** Full operator table (`eq`, `!eq`, `ieq`, `like`, `!like`, `ilike`, `!ilike`, `$like`, `like$`, `in`, `!in`, `gt`, `ge`, `lt`, `le`, `null`, `!null`, `empty`, `!empty`, `token`, `!token`). AND vs OR (`rootJunction=OR`). Nested filters across references. Practical patterns: search-as-you-type, "show only items I can edit," filtering by sharing.
- **Ordering, paging, total counts.** `order=…:asc`, secondary order, `page`, `pageSize`, `paging=false` (with caveats), `totalPages=true` (and its cost).
- **Gist API.** `/api/<type>/gist`, `/api/<type>/{uid}/gist`, reference navigation `/api/users/<uid>/userGroups/gist`. What it's good for, what it doesn't support (smaller field/filter surface).
- **Cross-type queries.** `/api/metadata?dataElements=true&programs=true&fields=…` for snapshots. `/api/identifiableObjects/{uid}` to resolve unknown UIDs.

Each subsection has at least one verified, runnable example. Length target: ~400 lines.

### `references/metadata/schemas.md`

Short, focused. Sections:

- **What `/api/schemas` is.** Returns a descriptor per metadata type: singular/plural names, endpoint, properties, references, `embeddedObject`, `persisted`, authorities, `shareable`.
- **`/api/schemas/<type>` and `/api/schemas/<type>/<property>`.** Reading one type or one property.
- **AI workflow.** When asked about an unfamiliar type, hit `/api/schemas/<type>` first. Use it to find required props (`required: true`), references (`propertyType: REFERENCE`, `klass`), collection props, embedded objects.
- **Schemas vs controller source.** When schemas is enough; when to fall back to the opensrc'd Java controllers (link to `data-fetching.md` step 1).
- **Worked example.** Discover what fields a `dataElement` accepts at runtime.

Length target: ~120 lines.

### `references/metadata/common-types.md`

Recipe per type. Each recipe: endpoint, key fields, required references, common gotchas, minimal create payload. Types covered:

1. **Organisation Units** — `parent`, `path`, `level`, `openingDate`, geometry, `organisationUnitGroups`. Gotcha: parent must exist before creation; level is derived from `path`.
2. **Data Elements** — `valueType`, `aggregationType`, `domainType` (AGGREGATE vs TRACKER), `categoryCombo` (default UID), `optionSet`. Gotcha: AGGREGATE belongs in datasets, TRACKER in program stages — different validity rules.
3. **Indicators** — `numerator` / `denominator` expressions, `indicatorType`, `annualized`. Gotcha: expressions use `#{deUid}` and `#{deUid.cocUid}` syntax; reference UIDs validated server-side.
4. **Programs** — `programType` (`WITH_REGISTRATION` = tracker, `WITHOUT_REGISTRATION` = event), `trackedEntityType`, `programStages`, `programTrackedEntityAttributes`. Gotcha: tracker programs require a TET; event programs do not.
5. **Tracked Entity Types & Attributes** — TET owns TEAs; programs link TEAs via `programTrackedEntityAttributes`. Gotcha: uniqueness is set on the TEA + program scope, not the TET.
6. **Option Sets & Options** — `valueType`, ordered `options` collection (`code`, `name`, `sortOrder`). Gotcha: options delete with the set; option `code` is the stored value and case-sensitive.
7. **Category Combos / Categories / Category Options** — the disaggregation triple. The "default" combo is what most things use. Gotcha: changing `categoryCombo` on a data element in use is destructive.

Each recipe is ~15-25 lines. Length target: ~300 lines total.

### `references/metadata/sharing.md`

Short. Sections:

- **The sharing model.** `publicAccess`, `userAccesses[]`, `userGroupAccesses[]`, `externalAccess`, owner (`createdBy` / `user`).
- **Access string format.** 8-char access string (positions 1-2 metadata read/write, 3-4 data read/write, 5-8 reserved — exact semantics confirmed during the verification pass). Cheat sheet of common combinations (`rwrw----`, `r-r-----`, `--------`, etc.). Note the 2.36 change from 5-char to 8-char and the `useFeature` pattern for handling it.
- **Read sharing.** `GET /api/sharing?type=<singularSchemaName>&id=<uid>`. The `type` uses the singular schema name (`dataElement`, not `dataElements`).
- **Write sharing.** `POST /api/sharing?type=…&id=…` with the same shape.
- **What's shareable.** Schemas tell you (`shareable: true`); embedded objects inherit from parent.
- **Common patterns.** Public read-only; share with a user group RW; copy sharing from another object; remove all user/group accesses; locking down to owner only.

Length target: ~150 lines.

## Verification plan

Before writing content, a ~10-minute grounding pass:

1. **docs.dhis2.org** — WebFetch the 2.42 docs for metadata overview, field filter, object filter, Gist API, schemas, sharing. Establish the canonical reference.
2. **Live play instance** — Playwright against `play.im.dhis2.org/stable-2-42` (or current published stable equivalent) for: `/api/schemas`, a sample `/api/dataElements?fields=…&filter=…`, `/api/<type>/gist`, `/api/sharing` GET. Verify response shapes and parameter behavior.
3. **opensrc'd source** — Spot-check 2-3 controllers for ambiguous behavior (gist parser, sharing access-string parser).

Any quirks that contradict the doc become "gotcha" callouts in the relevant file.

## Quality bar

- Every endpoint and parameter mentioned is verified against either docs.dhis2.org/2.42 or the live play instance (preferably both).
- Every code example is syntactically valid and uses real metadata properties — no invented fields.
- Examples cap at ~25 lines; longer recipes get split.
- Existing skill conventions (React 18, `@dhis2/ui`, CSS modules, `i18n.t`, `displayName` over `name`, `useApiDataQuery` hook pattern) are honored throughout.

## Open questions

None — scope, structure, depth, version, and verification approach are all settled.

## Implementation order (for the plan)

1. Verification pass (docs + live play + spot-check source).
2. Write `references/metadata.md` (entry — frames everything else).
3. Write `references/metadata/querying.md` (most cross-cutting).
4. Write `references/metadata/schemas.md` (referenced by every other file).
5. Write `references/metadata/common-types.md` (depends on querying + schemas being settled).
6. Write `references/metadata/sharing.md`.
7. Update `SKILL.md` (routing rows + rule).
8. Cross-link verification: every `→` reference between files resolves to the right anchor.
