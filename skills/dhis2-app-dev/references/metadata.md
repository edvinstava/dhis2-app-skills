# DHIS2 Metadata

## Metadata vs data

Metadata defines the structure of a DHIS2 instance — organisation units, programs, data
elements, option sets, and similar configuration objects. It changes rarely during a user
session, so fetch it once and cache aggressively (`staleTime: Infinity`). Data (tracked
entities, events, data values, analytics results) changes frequently and needs shorter
cache windows. See [data-fetching.md](./data-fetching.md) for the full caching strategy.

---

## The metadata domain map

The core types form a graph of compositional and referential relationships. Organisation
units are hierarchical nodes that ultimately hold data values. Data elements are either
`AGGREGATE` (grouped into data sets for periodic reporting) or `TRACKER` (attached to
program stages for event/tracker capture). Both use category combos to express
disaggregation dimensions; category combos are built from categories, which in turn hold
category options. When a data element restricts its allowed values to a finite list, it
references an option set. Indicators compute derived values by referencing data elements
in numerator/denominator expressions. Programs are either `WITHOUT_REGISTRATION` (event programs, single-event, no enrollment) or
`WITH_REGISTRATION` (tracker, with enrollment). Both own program stages,
which own program stage data elements. Tracker programs additionally link a tracked entity
type and tracked entity attributes via program tracked entity attributes. Data sets group
data elements for aggregate data entry.

Relationship summary:

| Type | Owned by / relates to |
|------|-----------------------|
| `organisationUnit` | Hierarchical (parent/children); root = system boundary |
| `dataSet` | Groups `dataElement` objects for aggregate reporting |
| `dataElement` | Has one `categoryCombo`; optionally has an `optionSet`; `domainType`: `AGGREGATE` or `TRACKER` |
| `categoryCombo` | Composed of `category` objects; each `category` has `categoryOption` objects |
| `optionSet` | Holds a list of `option` objects |
| `indicator` | References `dataElement` UIDs inside numerator/denominator expressions |
| `program` | Has `programType` (`WITHOUT_REGISTRATION` or `WITH_REGISTRATION`); owns `programStage` objects |
| `programStage` | Owned by `program`; owns `programStageDataElement` objects (links to `dataElement`) |
| `trackedEntityType` | Linked to a tracker `program` via `programTrackedEntityAttribute` |
| `trackedEntityAttribute` | Linked to `program` via `programTrackedEntityAttribute`; linked to `trackedEntityType` via `trackedEntityTypeAttribute` |

---

## Identifiers

DHIS2 metadata objects are addressed by **UID** — an opaque 11-character identifier
matching `/^[A-Za-z][A-Za-z0-9]{10}$/`. Generated server-side; treat as opaque.

Confirmed in `CodeGenerator.java`:
```
public static final String UID_REGEXP = "^[a-zA-Z][a-zA-Z0-9]{10}$";
```

Each object also carries:

- `code` — optional, user-defined, intended to be human-readable and stable across
  instances. Use for cross-instance references when UIDs differ.
- `name` — the source name in the object's authoring locale. Required on most types.
- `shortName` — abbreviated label for narrow contexts (tables, charts).
- `displayName` — `name` resolved through the user's active translation. **Always
  prefer `displayName` in UIs** — `name` shows the wrong language to non-default users.
- `displayShortName` — same, for `shortName`.
- `translations` — array of `{ locale, property, value }` triples driving `displayName` /
  `displayShortName`. The `property` value uses screaming snake case: `"NAME"`,
  `"SHORT_NAME"`, `"DESCRIPTION"`, etc.
- `href` — server-rendered absolute URL to the object's endpoint.

`displayName` resolution: `BaseIdentifiableObject.getDisplayName()` calls
`getTranslation("NAME", getName())` — it returns the translation for the user's active
locale if one exists, falling back to `name` when no translation is found. This is
computed on the server; it is not a persisted field and cannot be written.

---

## /api/metadata vs per-type endpoints

**Per-type endpoints** (`/api/dataElements`, `/api/programs`, `/api/organisationUnits`,
etc.) are the normal route for reading, creating, and updating individual objects or
paginated lists. They support field selection, filtering, ordering, and paging. Use
these for any CRUD operation and for all list or detail views in an app.

**`/api/metadata`** is a bulk read/write endpoint. A `GET` request returns a snapshot of
all (or selected) metadata types in a single response. A `POST` request accepts a payload
of multiple types simultaneously and can import an entire configuration in one round trip.
It is useful for configuration snapshots, cross-instance migrations, and dependency
exports. Bulk import details (importStrategy, atomicMode, mergeMode) are deferred — see
the Out of scope section below.

---

## When to read which sub-doc

| Task | Read |
|------|------|
| Build a query, filter, or list view | [`metadata/querying.md`](./metadata/querying.md) |
| Discover the shape of a type at runtime | [`metadata/schemas.md`](./metadata/schemas.md) |
| Create or edit a specific metadata type | [`metadata/common-types.md`](./metadata/common-types.md) |
| Set or change sharing | [`metadata/sharing.md`](./metadata/sharing.md) |

---

## Out of scope (deferred)

Bulk import via `POST /api/metadata` (with `importStrategy`, `atomicMode`, and
`mergeMode` parameters), dependency export (pulling an object and all its referenced dependencies via `/api/metadata`),
and metadata versioning/sync (the
`/api/metadata/version` subsystem) are not covered here. These topics would be added
to this file or to dedicated sub-docs if they become relevant.
