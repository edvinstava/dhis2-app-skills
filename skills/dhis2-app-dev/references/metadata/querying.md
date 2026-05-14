# Querying DHIS2 Metadata

DHIS2's metadata endpoints share a common query surface across all types — the same
`fields`, `filter`, `order`, and `paging` parameters work on `/api/dataElements`,
`/api/programs`, `/api/organisationUnits`, and every other per-type endpoint. This
file covers that surface in full. Before writing any query, read the controller source
for the version you're targeting (see [data-fetching.md](../data-fetching.md) step 1) —
parameter behavior can vary between versions.

---

## Field selection (`fields`)

Always include a `fields` parameter. Without it the server returns every persisted field
on the object, which is wasteful and slow. Specify only what the component actually
renders.

### Basic and nested fields

Comma-separate top-level fields. For reference and collection fields, expand sub-fields
inside brackets:

```
GET /api/dataElements.json?fields=id,displayName&pageSize=2
```

Response excerpt:
```json
{
  "dataElements": [
    { "id": "FTRrcoaog83", "displayName": "Accute Flaccid Paralysis (Deaths < 5 yrs)" },
    { "id": "P3jJH5Tu5VC", "displayName": "Acute Flaccid Paralysis (AFP) follow-up" }
  ]
}
```

Nested expansion with brackets — brackets can nest to arbitrary depth:

```
GET /api/dataElements.json?fields=id,displayName,categoryCombo[id,displayName,categories[id,displayName]]&pageSize=2
```

Response excerpt:
```json
{
  "dataElements": [
    {
      "id": "FTRrcoaog83",
      "displayName": "Accute Flaccid Paralysis (Deaths < 5 yrs)",
      "categoryCombo": {
        "id": "bjDvmb4bfuf",
        "displayName": "default",
        "categories": [{ "id": "GLevLNI9wkl", "displayName": "default" }]
      }
    }
  ]
}
```

### Field presets

Use presets as a starting point, not as a replacement for explicit field lists. The server
computes them at query time. All five are verified against the 2.42 play instance.

| Preset | What it returns |
|--------|-----------------|
| `:identifiable` | `id`, `code`, `name`, `created`, `lastUpdated`, `lastUpdatedBy` — the minimum to identify an object |
| `:nameable` | `id`, `code`, `name`, `shortName`, `description`, `created`, `lastUpdated`. Note: not a superset of `:identifiable` — drops `lastUpdatedBy`. |
| `:simple` | Non-relational scalar fields plus display fields: `id`, `code`, `name`, `shortName`, `displayName`, `displayShortName`, `displayFormName`, `created`, `lastUpdated`, `aggregationType`, `valueType`, `domainType`, etc. (varies by type) |
| `:owner` | Properties flagged as `owner: true` in the schema — typically the canonical writable surface for create/update payloads. To know exactly which fields, check `/api/schemas/<type>` (see schemas.md). |
| `:all` | Every field including computed and non-persisted (`access`, `favorites`, `href`, `favorite`). Much larger response than `:owner`. Rarely needed in production. |

Example — `:identifiable` on data elements returns `id`, `code`, `name`, `created`,
`lastUpdated`, `lastUpdatedBy` (no `displayName`):

```
GET /api/dataElements.json?fields=:identifiable&pageSize=2
```

### Field transforms

Transforms modify a field before it reaches the response. They do not filter rows.

| Syntax | Effect |
|--------|--------|
| `field~rename(alias)` | Renames the field key in the response |
| `collection::size` | Returns the count of collection items as an integer instead of the collection itself |
| `collection::isNotEmpty` | Returns a boolean instead of the collection |
| `!field` | Excludes the field from a preset expansion |

**Rename example** — useful when the client expects a different key name:

```
GET /api/dataElements.json?fields=displayName~rename(label),id&pageSize=2
```

Response excerpt:
```json
{
  "dataElements": [
    { "id": "FTRrcoaog83", "label": "Accute Flaccid Paralysis (Deaths < 5 yrs)" },
    { "id": "P3jJH5Tu5VC", "label": "Acute Flaccid Paralysis (AFP) follow-up" }
  ]
}
```

**`::size` example** — count how many data sets each element belongs to without loading
the full collection:

```
GET /api/dataElements.json?fields=id,displayName,dataSetElements::size&pageSize=2
```

Response excerpt:
```json
{
  "dataElements": [
    { "id": "FTRrcoaog83", "displayName": "Accute Flaccid Paralysis (Deaths < 5 yrs)", "dataSetElements": 3 },
    { "id": "P3jJH5Tu5VC", "displayName": "Acute Flaccid Paralysis (AFP) follow-up", "dataSetElements": 1 }
  ]
}
```

**Exclusion example** — start from `:identifiable` but drop `code`:

```
GET /api/dataElements.json?fields=:identifiable,!code&pageSize=2
```

Returns `id`, `name`, `created`, `lastUpdated`, `lastUpdatedBy` — `code` is absent.

**Nested collection pagination via field transform is not available in 2.42.** A
`~paging(page,pageSize)` transform syntax appears in some older DHIS2 documentation,
but the transform is not implemented in 2.42 — the server returns a 500 Internal Server
Error when the syntax is used (verified against `stable-2-42-4-1`). To page a nested
collection, use the gist collection navigation endpoint
(`/api/<type>/<uid>/<collection>/gist`) instead.

---

## Filtering (`filter`)

The `filter` parameter takes the form `property:operator:value`. Multiple `filter`
parameters are AND-ed by default; `rootJunction=OR` switches to OR logic.

### Operator table

Operators are verified from `DefaultQueryParser.java` in the 2.42 source. Operators not
listed here do not exist in 2.42 — for example, `between` has an implementation class
but is not wired into the query parser and returns an error if used.

| Operator | Meaning | Example |
|----------|---------|---------|
| `eq` | Equals (case-sensitive) | `filter=valueType:eq:NUMBER` |
| `!eq` / `neq` / `ne` | Not equals (three aliases) | `filter=valueType:!eq:TEXT` |
| `ieq` | Equals (case-insensitive) | `filter=code:ieq:de_123` |
| `gt` | Greater than | `filter=created:gt:2020-01-01` |
| `ge` / `gte` | Greater than or equal (two aliases) | `filter=lastUpdated:ge:2023-01-01` |
| `lt` | Less than | `filter=sortOrder:lt:10` |
| `le` / `lte` | Less than or equal (two aliases) | `filter=sortOrder:le:5` |
| `like` | Substring, case-sensitive | `filter=name:like:Vacc` |
| `!like` | Not substring, case-sensitive | `filter=name:!like:test` |
| `$like` | Starts with, case-sensitive | `filter=name:$like:ANC` |
| `!$like` | Not starts with, case-sensitive | `filter=name:!$like:ANC` |
| `like$` | Ends with, case-sensitive | `filter=name:like$:visit` |
| `!like$` | Not ends with, case-sensitive | `filter=name:!like$:test` |
| `ilike` | Substring, case-insensitive | `filter=displayName:ilike:malaria` |
| `!ilike` | Not substring, case-insensitive | `filter=displayName:!ilike:draft` |
| `$ilike` / `startsWith` | Starts with, case-insensitive (two aliases) | `filter=displayName:$ilike:anc` |
| `!$ilike` | Not starts with, case-insensitive | `filter=displayName:!$ilike:anc` |
| `ilike$` / `endsWith` | Ends with, case-insensitive (two aliases) | `filter=displayName:ilike$:visit` |
| `!ilike$` | Not ends with, case-insensitive | `filter=displayName:!ilike$:draft` |
| `token` | Token/word search (matches word boundaries) | `filter=displayName:token:malaria` |
| `!token` | Not token search | `filter=displayName:!token:draft` |
| `in` | Value is in the list (bracket syntax) | `filter=valueType:in:[NUMBER,INTEGER]` |
| `!in` | Value is not in the list | `filter=valueType:!in:[TEXT,LONG_TEXT]` |
| `null` | Field is null (no value needed) | `filter=code:null` |
| `!null` | Field is not null (no value needed) | `filter=code:!null` |
| `empty` | Collection is empty (no value needed) | `filter=dataSetElements:empty` |
| `!empty` | Collection is not empty (no value needed) | `filter=userGroups:!empty` |

Two source discrepancies worth knowing: `startsWith` and `endsWith` are valid aliases for
`$ilike` and `ilike$` — the docs don't mention them but the source supports them. The
`gte` and `lte` aliases for `ge`/`le` also work in the source but are not in the docs.

### AND filtering (default)

Multiple `filter` parameters are AND-ed. This query returns aggregate data elements whose
`valueType` is `NUMBER` or `INTEGER`:

```
GET /api/dataElements.json?filter=valueType:in:[NUMBER,INTEGER]&filter=domainType:eq:AGGREGATE&fields=id,displayName,valueType,domainType&pageSize=5
```

Response excerpt (481 total results):
```json
{
  "dataElements": [
    { "id": "FTRrcoaog83", "displayName": "Accute Flaccid Paralysis (Deaths < 5 yrs)", "valueType": "NUMBER", "domainType": "AGGREGATE" },
    { "id": "P3jJH5Tu5VC", "displayName": "Acute Flaccid Paralysis (AFP) follow-up", "valueType": "NUMBER", "domainType": "AGGREGATE" }
  ]
}
```

### OR filtering (`rootJunction=OR`)

Add `rootJunction=OR` to match rows where any filter matches instead of all. This returns
elements that match `displayName:ilike:bcg` OR have `valueType:eq:NUMBER`:

```
GET /api/dataElements.json?filter=displayName:ilike:bcg&filter=valueType:eq:NUMBER&rootJunction=OR&fields=id,displayName,valueType&pageSize=5
```

`rootJunction=OR` applies to the entire filter set — there is no per-filter grouping syntax.
If you need mixed AND/OR logic, split into separate requests or use a different approach.

### Nested filters across references

Filter on a property of a related object using dot notation. This returns programs that
have at least one stage data element with `valueType=NUMBER`:

```
GET /api/programs.json?filter=programStages.programStageDataElements.dataElement.valueType:eq:NUMBER&fields=id,displayName&pageSize=3
```

Response excerpt (7 total results):
```json
{
  "programs": [
    { "id": "lxAQ7Zs9VYR", "displayName": "Antenatal care visit" },
    { "id": "IpHINAT79UW", "displayName": "Child Programme" },
    { "id": "eBAyeGv0exc", "displayName": "Inpatient morbidity and mortality" }
  ]
}
```

Nested filter paths follow the Java property chain from the root object. Read the schema
at `/api/schemas/<type>` to find valid property names before building a nested path — the
server returns a 400 if any path segment is unresolvable.

### Practical patterns

#### Search-as-you-type

Build the filter array dynamically — include the search filter only when the user has
typed something. Pass the array directly to `useApiDataQuery` params:

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

The `@dhis2/app-runtime` data engine serialises an array of strings to repeated `filter`
query parameters, which the server AND-s together. When `searchTerm` is empty the
spread produces no element and that filter is omitted entirely.

#### Filtering by enumerated constants

`valueType` and `domainType` are `CONSTANT` properties with a fixed list of values
(visible in `/api/schemas/dataElement.json`). Use `eq` for a single value or `in` for
a list:

```
GET /api/dataElements.json?filter=valueType:in:[NUMBER,INTEGER]&filter=domainType:eq:AGGREGATE&fields=id,displayName,valueType,domainType&pageSize=5
```

The bracket syntax for `in` requires no spaces: `[NUMBER,INTEGER]`, not
`[NUMBER, INTEGER]`.

---

## Ordering, paging, total counts

### Ordering

The `order` parameter takes the form `property:asc` or `property:desc`. Pass multiple
`order` parameters for secondary sort — the server applies them left to right:

```
GET /api/dataElements.json?order=displayName:asc&order=created:desc&fields=id,displayName,created&pageSize=3
```

Multiple `order` params are confirmed to work in 2.42 — the server accepts repeated
`order` query parameters (not a comma-separated string).

### Paging

By default, responses are paginated with `pageSize=50`. The `pager` object in every
paginated response contains `page`, `pageSize`, `total`, and `nextPage` (when there are
more pages):

```json
{
  "pager": {
    "page": 1,
    "total": 1037,
    "pageSize": 2,
    "pageCount": 519,
    "nextPage": "https://..."
  }
}
```

Control paging with `page` and `pageSize`:

```typescript
params: {
    page: currentPage,
    pageSize: 50,
    fields: 'id,displayName',
}
```

### `paging=false`

Setting `paging=false` disables pagination and returns all matching objects in a single
response with no `pager` envelope. Use with caution — on a production instance,
`/api/dataElements?paging=false` returns all 1037+ data elements in a single JSON payload.
Reserve this for small metadata types (option sets, categories) or for offline/export
scenarios where you genuinely need the full list.

### `totalPages=true`

On standard per-type metadata endpoints (`/api/dataElements`, `/api/programs`, etc.) the
`pager` always includes `total` and `pageCount` — no extra parameter is needed. Verified
against 2.42: `?pageSize=2&fields=id` and `?pageSize=2&fields=id&totalPages=true` return
an identical pager.

The `totalPages=true` parameter has no visible effect on these endpoints. Its main
relevance is documented in the Gist section: the gist API omits `total` by default, and
the guidance for that case is to use the regular endpoint instead.

---

## The Gist API

`/api/<type>/gist` is a lighter, HQL-backed endpoint optimised for large lists. It returns
a compact response with fewer default fields and renders reference fields as UIDs rather
than nested objects.

### Basic usage

```
GET /api/dataElements/gist.json?fields=id,displayName&pageSize=3
```

Response:
```json
{
  "pager": {
    "page": 1,
    "pageSize": 3,
    "nextPage": "http://..."
  },
  "dataElements": [
    { "id": "pikOziyCXbM", "displayName": "OPV1 doses given" },
    { "id": "O05mAByOgAv", "displayName": "OPV2 doses given" },
    { "id": "vI2csg55S9C", "displayName": "OPV3 doses given" }
  ]
}
```

Notice: the gist pager does not include `total` by default.

### Reference fields render as UIDs

When you include a reference field (like `categoryCombo`) in a gist request, it renders
as a UID string and the response includes an `apiEndpoints` map for navigation:

```
GET /api/dataElements/gist.json?fields=id,displayName,categoryCombo&pageSize=2
```

Response:
```json
{
  "dataElements": [
    {
      "id": "pikOziyCXbM",
      "displayName": "OPV1 doses given",
      "categoryCombo": "dzjKKQq0cSO",
      "apiEndpoints": {
        "categoryCombo": "/api/categoryCombos/dzjKKQq0cSO/gist"
      }
    }
  ]
}
```

### Collection navigation

Gist supports walking associations at `/api/<type>/<uid>/<collection>/gist`. This is the
primary use case — navigate a user's group memberships without loading the full user
object:

```
GET /api/users/oXD88WWSQpR/userGroups/gist.json?pageSize=3
```

The server returns the associated `userGroup` objects with gist-style default fields
plus `apiEndpoints` for further navigation.

### Gotchas

**Nested non-persisted fields fail with 400.** The gist API uses HQL directly, so it can
only address fields that are mapped Hibernate properties on the entity. `displayName` is a
computed (non-persisted) field on referenced objects. This request fails:

```
GET /api/dataElements/gist.json?fields=id,displayName,categoryCombo[id,displayName]
```

Error:
```
could not resolve property: displayName of: org.hisp.dhis.category.CategoryCombo
```

The regular endpoint handles this transparently via its field-transformer layer; gist does
not. To fetch a reference's display name via gist, follow the `apiEndpoints` URL returned
for that reference.

**The pager omits `total` by default.** If your UI needs to show "X results", use the
regular endpoint with `totalPages=true`, or accept that gist will not tell you the total.

**`displayName` works on the root entity.** The limitation above only applies to
*nested* reference fields. `displayName` on the root type (e.g.
`fields=id,displayName` on `/api/dataElements/gist`) works fine because HQL can address
it on the queried entity.

### When to prefer gist

Use gist when:
- You need a fast, lightweight list of a large collection and don't need nested object data.
- You're navigating associations (e.g., all user groups a user belongs to).
- You don't need `total` in the pager.

Use the regular endpoint when:
- You need nested field expansion (e.g., `categoryCombo[id,displayName]`).
- You need `total` or `pageCount` in the pager.
- You need the full field-transformer surface (presets, `::size`, `~rename`).

---

## Cross-type queries

### `/api/metadata` bulk read

Fetch multiple types in a single request by passing type flags as query parameters:

```
GET /api/metadata.json?dataElements=true&programs=true&fields=id
```

The response is an object keyed by the plural type name, plus a `system` metadata block:

```json
{
  "system": {
    "id": "eed3d451-...",
    "version": "2.42.4.1",
    "date": "2026-05-14T09:41:14.746+0000"
  },
  "dataElements": [ { "id": "..." }, ... ],
  "programs": [ { "id": "..." }, ... ]
}
```

The `fields` parameter applies to all requested types. This is useful for configuration
snapshots (export all metadata of interest in one round trip) or for seeding an offline
store. Be cautious with `paging=false` here — on a populated instance, a multi-type
request without field restriction can be a very large payload.

### `/api/identifiableObjects/{uid}` — resolve any UID

If you have a UID but don't know its type, use this endpoint:

```
GET /api/identifiableObjects/FTRrcoaog83.json?fields=id,displayName,href
```

Response:
```json
{
  "id": "FTRrcoaog83",
  "displayName": "Accute Flaccid Paralysis (Deaths < 5 yrs)",
  "href": "https://.../api/dataElements/FTRrcoaog83"
}
```

The response is the full object at its native type endpoint — identical to calling
`/api/dataElements/FTRrcoaog83` directly. The `href` field in the response points to
the type-specific URL, so the caller can discover the type from it. The `fields`
parameter works here as on any endpoint; omitting it returns all fields.

This is useful in audit logs, external links, and search results where you're given a UID
without context — resolve once and redirect to the correct per-type endpoint.
