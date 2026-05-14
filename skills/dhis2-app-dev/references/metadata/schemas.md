# Self-discovering metadata types with /api/schemas

`/api/schemas` is the runtime contract for DHIS2's metadata model. Before composing
a query or a create/update payload for an unfamiliar type, hit this endpoint — it tells
you every field, its type, and whether it is required, writable, and persisted. Do not
guess field names from training data.

---

## What /api/schemas returns

A GET to `/api/schemas` returns a `schemas` array with one descriptor per metadata
type. The 2.42 play instance has **119 entries** in total (105 persisted, 89 with a
`relativeApiEndpoint`). Types without a `relativeApiEndpoint` are embedded objects
(e.g. `trackedEntityTypeAttribute`, `sharing`) — they exist as sub-documents of other
types and have no standalone endpoint.

Key top-level descriptor fields:

| Field | Meaning |
|-------|---------|
| `klass` | Fully-qualified Java class name (unique type identity) |
| `singular` | Singular name — used in `GET /api/schemas/<singular>` and in `type` params (e.g. `/api/sharing?type=dataElement`) |
| `plural` | Plural name — the resource path segment (`/api/dataElements`) |
| `name` | Display name for the schema (usually equals `singular`; a few types use a different string) |
| `relativeApiEndpoint` | Path relative to `/api` — e.g. `/dataElements`. Absent on embedded objects |
| `persisted` | `true` if the type is stored in the database |
| `shareable` | `true` if objects of this type can have `publicAccess` / user/group sharing |
| `dataShareable` | `true` if positions 3-4 of the access string (data read/write) apply to this type |
| `embeddedObject` | `true` if this type is always embedded inside another (no standalone endpoint) |
| `identifiableObject` | `true` if objects carry an `id` UID field |
| `authorities` | Required user authorities for CREATE_PUBLIC, CREATE_PRIVATE, and DELETE operations |
| `properties` | Array of property descriptors — one per field on the type |

Narrow the response with `fields` — the schema endpoint participates in the standard
field-filter system:

```
GET /api/schemas.json?fields=singular,plural,relativeApiEndpoint,shareable,dataShareable
```

---

## Per-type and per-property endpoints

**Single type:**

```
GET /api/schemas/dataElement.json
```

Returns the full schema descriptor for `dataElement` (top-level fields + full
`properties[]` array). Supports `fields` filtering:

```
GET /api/schemas/dataElement.json?fields=properties[name,fieldName,propertyType,klass,required,writable,persisted,itemKlass]
```

**Single property on a type:**

```
GET /api/schemas/dataElement/categoryCombo
```

Returns the property descriptor for `categoryCombo` on `dataElement` — confirmed
in `SchemaController.java` (`GET /{type}/{property}`). Note: there is no
`/api/schemas/{type}/properties` listing route. To list all properties, use
`GET /api/schemas/{type}` (with a `fields=properties[...]` filter if needed).

---

## The AI workflow

When asked to query or mutate an unfamiliar metadata type, start here:

1. **Fetch the schema.**

   ```
   GET /api/schemas/<singular>.json?fields=properties[name,fieldName,propertyType,klass,itemKlass,itemPropertyType,required,writable,persisted,embeddedObject]
   ```

2. **Read `properties[]`.**

   - **Required create fields:** `required: true` AND `writable: true` AND
     `persisted: true` — these must be in every create payload.
   - **Reference fields:** `propertyType: "REFERENCE"` — the `klass` field names
     the target type. Pass the referenced object as `{"id": "<uid>"}`.
   - **Collection fields:** `propertyType: "COLLECTION"` — `itemPropertyType`
     tells you whether items are references (`"REFERENCE"`) or scalar. For
     reference collections, each item is `{"id": "<uid>"}`.
   - **Embedded objects:** `embeddedObject: true` — these are inlined sub-documents
     (e.g. `sharing`, `style`, `access`), not standalone references.
   - **Computed fields:** `persisted: false` — server-generated; never send these
     in a create/update payload (e.g. `displayName`, `href`, `access`).
   - **Enumeration fields:** `propertyType: "CONSTANT"` — the schema includes a
     `constants` array listing every valid value.

3. **Build the query or payload** from what you learned.

   For queries: only fields with `readable: true` can appear in a `fields` parameter.
   For create/update payloads: only include fields with `writable: true`.

---

## When schemas isn't enough

Schemas describe the **shape** — field names, types, and cardinality. They do not
describe:

- **Validation rules** — e.g. that `numerator` on an indicator must be a valid
  expression referencing existing UIDs, or that `aggregationType` interacts with
  `domainType`.
- **Cross-property constraints** — e.g. TRACKER data elements have different rules
  around `aggregationType` and `categoryCombo` than AGGREGATE ones.
- **Side effects** — e.g. changing `categoryCombo` on a data element that already
  has data values requires `force=true`.
- **Business logic defaults** — e.g. the server applies a sensible default
  `categoryCombo` if none is given (confirmed in 2.42), despite the schema marking
  `categoryCombo` as `required: true`.

For these, fall back to the controller source. Use `npx opensrc path dhis2/dhis2-core`
to get the cached source path, then search for the relevant controller (see
[data-fetching.md](../data-fetching.md) step 1 for the full workflow).

---

## Worked example: discover the writable surface of `dataElement`

```
GET /api/schemas/dataElement.json?fields=properties[name,fieldName,propertyType,klass,required,writable,persisted,itemKlass]
```

Returns (excerpt — the full response has ~50 properties):

```json
{
  "properties": [
    { "name": "name", "fieldName": "name", "propertyType": "TEXT", "required": true, "writable": true, "persisted": true },
    { "name": "shortName", "fieldName": "shortName", "propertyType": "TEXT", "required": true, "writable": true, "persisted": true },
    { "name": "valueType", "fieldName": "valueType", "propertyType": "CONSTANT", "klass": "org.hisp.dhis.common.ValueType", "required": true, "writable": true, "persisted": true },
    { "name": "aggregationType", "fieldName": "aggregationType", "propertyType": "CONSTANT", "klass": "org.hisp.dhis.analytics.AggregationType", "required": true, "writable": true, "persisted": true },
    { "name": "domainType", "fieldName": "domainType", "propertyType": "CONSTANT", "klass": "org.hisp.dhis.dataelement.DataElementDomain", "required": true, "writable": true, "persisted": true },
    { "name": "categoryCombo", "fieldName": "categoryCombo", "propertyType": "REFERENCE", "klass": "org.hisp.dhis.category.CategoryCombo", "required": true, "writable": true, "persisted": true },
    { "name": "zeroIsSignificant", "fieldName": "zeroIsSignificant", "propertyType": "BOOLEAN", "required": true, "writable": true, "persisted": true },
    { "name": "displayName", "fieldName": "displayName", "propertyType": "TEXT", "required": false, "writable": false, "persisted": false }
    /* ... */
  ]
}
```

Filter to `required: true && writable: true && persisted: true` and you have the
minimum create payload surface for a `dataElement`: `name`, `shortName`, `valueType`,
`aggregationType`, `domainType`, `categoryCombo`, `zeroIsSignificant`. All seven are
confirmed against the 2.42 verification snapshot.

Notice that `displayName` is `writable: false` and `persisted: false` — it is
computed server-side and must not appear in a create payload. The `categoryCombo`
property has `propertyType: "REFERENCE"`, so it is sent as `{"id": "<uid>"}`.
