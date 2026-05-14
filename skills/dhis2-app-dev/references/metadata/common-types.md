# Common metadata types — recipes

Concise recipes for the seven most-used metadata types. For an unfamiliar type, hit
`/api/schemas/<type>` (see [schemas.md](./schemas.md)) — these recipes follow the
same pattern you'd derive from there but call out the gotchas the schema alone won't
tell you.

---

## Organisation Units

**Endpoint:** `/api/organisationUnits` (schema singular: `organisationUnit`)

**Key fields:** `name`, `shortName`, `openingDate` (required), `parent` (reference to parent
org unit — omit only for the root), `path` (derived), `level` (derived), `geometry` (GeoJSON),
`code`, `description`, `closedDate`

**Required references:** `parent` must exist before creation (unless creating a root unit, which
requires special authority). There are no other reference prerequisites.

**Common gotchas:**
- `path` and `level` are derived server-side from `parent` — never include them in a create
  or update payload.
- `organisationUnit` is `shareable: false` in the schema (verified on 2.42) — there is no
  `/api/sharing` endpoint for org units. Access control is via the org-unit hierarchy assigned
  to users, not object-level sharing.
- `geometry` uses GeoJSON. Pass a standard GeoJSON `Point`, `Polygon`, or `MultiPolygon`
  object as the field value — the server stores it in the `geometry` column.
- `openingDate` is required. Use ISO 8601 date string (e.g. `"2024-01-01T00:00:00.000"`).

**Minimal create payload:**

```typescript
const payload = {
    name: 'Riverside Clinic',
    shortName: 'Riverside',
    openingDate: '2024-01-01T00:00:00.000',
    parent: { id: '<parent-org-unit-uid>' },
};

const createOrgUnitMutation = {
    resource: 'organisationUnits',
    type: 'create' as const,
    data: payload,
};
```

---

## Data Elements

**Endpoint:** `/api/dataElements` (schema singular: `dataElement`)

**Key fields:** `name`, `shortName`, `valueType`, `aggregationType`, `domainType`,
`categoryCombo` (reference), `zeroIsSignificant` (required boolean), `optionSet` (optional
reference), `description`, `code`, `formName`

**Required references:** `categoryCombo` is required in the schema. The server applies the
default category combo (`bjDvmb4bfuf`) if the field is omitted — but explicitly passing the
default UID is safer and makes the intent clear.

**Common gotchas:**
- `domainType: 'AGGREGATE'` elements participate in data sets for periodic reporting.
  `domainType: 'TRACKER'` elements participate in program stages. Different validation
  applies: TRACKER elements are not expected to have a meaningful `aggregationType`; the
  server accepts `NONE` for them.
- The "default" category combo UID is `bjDvmb4bfuf` — use it when the data element has
  no disaggregation.
- `zeroIsSignificant` is required (`required: true` in the schema). Pass `false` unless
  zero is a meaningful data value for this element.
- Changing `categoryCombo` on a data element that already has data values is destructive —
  the server will refuse without the `force=true` query parameter on the update request.
- `categoryCombo` is sent as a reference object `{ id: '...' }`, not a plain string.

**Minimal create payload:**

```typescript
const DEFAULT_CATEGORY_COMBO_UID = 'bjDvmb4bfuf';

const payload = {
    name: 'ANC 1st visit',
    shortName: 'ANC 1st visit',
    valueType: 'NUMBER',
    aggregationType: 'SUM',
    domainType: 'AGGREGATE',
    zeroIsSignificant: false,
    categoryCombo: { id: DEFAULT_CATEGORY_COMBO_UID },
};

const createDataElementMutation = {
    resource: 'dataElements',
    type: 'create' as const,
    data: payload,
};
```

---

## Indicators

**Endpoint:** `/api/indicators` (schema singular: `indicator`)

**Key fields:** `name`, `shortName`, `indicatorType` (reference), `numerator`,
`denominator`, `annualized` (required boolean), `numeratorDescription`,
`denominatorDescription`, `description`, `decimals`

**Required references:** `indicatorType` must exist — it defines the factor (e.g. per 1,
per 100, per 1000). Create or fetch one via `/api/indicatorTypes` before creating an
indicator.

**Common gotchas:**
- `numerator` and `denominator` use expression syntax: `#{dataElementUid}` for an
  aggregate data element total, `#{dataElementUid.categoryOptionComboUid}` for a specific
  disaggregation. The server validates that every UID referenced in the expression exists.
- Indicator values are computed at query time — they are not stored. Changing the
  expression changes what future analytics queries return; historical data is unaffected.
- `annualized` is required. Set it to `false` unless the indicator should be annualized
  (multiplied by 12 when the period is shorter than a year, e.g. for monthly data).
- `decimals` is optional and controls rounding in analytics output.

**Minimal create payload:**

```typescript
const payload = {
    name: 'ANC 1-3 Dropout Rate',
    shortName: 'ANC Dropout Rate',
    indicatorType: { id: '<indicator-type-uid>' },
    numerator: '#{fbfJHSPpUQD.pq2XI5kz2BY}',
    denominator: '#{fbfJHSPpUQD.pq2XI5kz2BY}',
    annualized: false,
};

const createIndicatorMutation = {
    resource: 'indicators',
    type: 'create' as const,
    data: payload,
};
```

---

## Programs (event and tracker)

**Endpoint:** `/api/programs` (schema singular: `program`)

**Key fields:** `name`, `shortName`, `programType`, `categoryCombo` (reference),
`skipOffline` (required boolean), `trackedEntityType` (reference — tracker programs only),
`programStages` (collection), `programTrackedEntityAttributes` (collection),
`organisationUnits`, `accessLevel`, `featureType`

**Required references:** `categoryCombo` is required. For tracker programs
(`programType: 'WITH_REGISTRATION'`), `trackedEntityType` must also be supplied. For event
programs (`programType: 'WITHOUT_REGISTRATION'`) no `trackedEntityType` is needed.

**Common gotchas:**
- `programType: 'WITH_REGISTRATION'` is a tracker program — it tracks enrollments of
  tracked entities and requires a `trackedEntityType`. `programType: 'WITHOUT_REGISTRATION'`
  is an event program — no enrollments, no TET, single-stage data capture.
- `skipOffline` is required (schema `required: true`). Pass `false` unless the program
  should be excluded from offline sync.
- `programStages` is owned by the program. Create stages after creating the program, or
  include them inline in the create payload with their full shape.
- `programTrackedEntityAttributes` links existing tracked entity attributes to this
  program. The TEAs themselves are global objects (`/api/trackedEntityAttributes`) — they
  are not created here, only referenced.
- Per-program uniqueness for a TEA is set on the `programTrackedEntityAttribute` link
  object (`unique: true`), not on the TEA itself.
- `organisationUnits` controls which facilities can use the program — typically populated
  after creating the program.

**Minimal create payload (event program):**

```typescript
const payload = {
    name: 'Malaria Case Notification',
    shortName: 'Malaria CN',
    programType: 'WITHOUT_REGISTRATION',
    categoryCombo: { id: 'bjDvmb4bfuf' },
    skipOffline: false,
};

const createProgramMutation = {
    resource: 'programs',
    type: 'create' as const,
    data: payload,
};
```

**Minimal create payload (tracker program):**

```typescript
const trackerPayload = {
    name: 'HIV Care',
    shortName: 'HIV Care',
    programType: 'WITH_REGISTRATION',
    categoryCombo: { id: 'bjDvmb4bfuf' },
    skipOffline: false,
    trackedEntityType: { id: '<tracked-entity-type-uid>' },
};
```

---

## Tracked Entity Types & Attributes

### Tracked Entity Types

**Endpoint:** `/api/trackedEntityTypes` (schema singular: `trackedEntityType`)

**Key fields:** `name`, `shortName`, `description`, `trackedEntityTypeAttributes`
(collection of association objects), `featureType`, `minAttributesRequiredToSearch`,
`allowAuditLog`

**Required references:** None — a TET has no required foreign keys.

**Common gotchas:**
- TEAs (tracked entity attributes) are global objects — they are not owned by or created
  inside a TET. `trackedEntityTypeAttributes` on a TET is an association list linking the
  TET to existing TEAs.
- The `trackedEntityTypeAttribute` association object (no standalone endpoint — it is an
  embedded object) contains: `trackedEntityAttribute: { id }`, `displayName`, `mandatory`,
  `searchable`. Set `searchable: true` on the attributes you want indexed for search.
- TEAs used for cross-TET search (e.g. a national ID shared across programs) should be
  created once globally and linked to multiple TETs.

**Minimal create payload:**

```typescript
const tetPayload = {
    name: 'Person',
    shortName: 'Person',
};

const createTETMutation = {
    resource: 'trackedEntityTypes',
    type: 'create' as const,
    data: tetPayload,
};
```

### Tracked Entity Attributes

**Endpoint:** `/api/trackedEntityAttributes` (schema singular: `trackedEntityAttribute`)

**Key fields:** `name`, `shortName`, `valueType`, `aggregationType` (required), `unique`,
`confidential`, `generated`, `optionSet` (optional reference), `pattern` (for generated
attributes)

**Required references:** None — TEAs are standalone objects.

**Common gotchas:**
- `aggregationType` is required (schema `required: true`). Use `NONE` for non-aggregate
  attributes (the common case for tracker).
- `unique: true` makes the attribute globally unique across all tracked entities of any
  type. For per-program uniqueness, set `unique: true` on the `programTrackedEntityAttribute`
  link instead, not here.
- `generated: true` enables server-side pattern-based ID generation using `pattern`. When
  `generated` is `true` the attribute value is auto-assigned and read-only.

**Minimal create payload:**

```typescript
const teaPayload = {
    name: 'National ID',
    shortName: 'National ID',
    valueType: 'TEXT',
    aggregationType: 'NONE',
};

const createTEAMutation = {
    resource: 'trackedEntityAttributes',
    type: 'create' as const,
    data: teaPayload,
};
```

---

## Option Sets & Options

**Endpoint:** `/api/optionSets` (schema singular: `optionSet`)

**Key fields:** `name`, `valueType` (required — constrains valid option `code` types),
`options` (inline or by reference), `version`

**Required references:** None — option sets are standalone.

**Common gotchas:**
- `valueType` on the option set constrains what kind of values the options' `code` fields
  can hold (e.g. `TEXT`, `INTEGER`, `DATE`). Most option sets use `TEXT`.
- Options are owned by the set — deleting the option set cascades to its options.
- `code` on an option is the persisted value stored in data values. It is case-sensitive.
  The `name` is what displays in the UI; the `code` is what gets stored. These can differ.
- `sortOrder` is 1-indexed and controls the display order. It is optional but should be
  set when order matters.
- Options can be created inline in the option set payload (recommended for small sets) or
  added individually via `POST /api/options` with `optionSet: { id }` as a reference.

**Minimal create payload:**

```typescript
const optionSetPayload = {
    name: 'Disease classification',
    valueType: 'TEXT',
    options: [
        { code: 'MAL', name: 'Malaria', sortOrder: 1 },
        { code: 'TUB', name: 'Tuberculosis', sortOrder: 2 },
        { code: 'HIV', name: 'HIV/AIDS', sortOrder: 3 },
    ],
};

const createOptionSetMutation = {
    resource: 'optionSets',
    type: 'create' as const,
    data: optionSetPayload,
};
```

---

## Category Combos, Categories, Category Options

**Endpoint:** `/api/categoryCombos` (schema singular: `categoryCombo`)

**Key fields:** `name`, `dataDimensionType` (required — `DISAGGREGATION` or `ATTRIBUTE`),
`skipTotal` (required boolean), `categories` (collection of references)

**Required references:** `categories` must exist before creating a combo. Create
`categoryOptions` first, then `categories` that reference them, then the `categoryCombo`
that references the categories.

**Common gotchas:**
- The creation order is: `categoryOptions` → `categories` (each references its options) →
  `categoryCombo` (references its categories). Skipping this order causes reference errors.
- `categoryOptionCombos` (note the spelling — plural with an extra "O") are the
  materialized cross-product of the combo's categories and their options. They are
  auto-generated server-side when the category combo is saved — do not include them in a
  create payload.
- The "default" category combo (UID `bjDvmb4bfuf`) is what most data elements and programs
  use when there is no disaggregation. Never delete or modify it.
- `dataDimensionType: 'DISAGGREGATION'` is the standard choice for data element
  disaggregation. `'ATTRIBUTE'` is for program attribute dimensions.
- `skipTotal` controls whether the "total" category option combo is generated for the
  cross-product. Set to `false` for normal disaggregation combos.
- Changing `categoryCombo` on a data element that already has data values requires
  `force=true` as a query parameter on the update request: `PUT /api/dataElements/<uid>?force=true`.
  Without `force=true` the server rejects the change.

**Minimal create payload:**

```typescript
// Step 1: create category options (one per disaggregation value)
// Step 2: create a category referencing those options
// Step 3: create the combo
const categoryComboPayload = {
    name: 'Age and Sex',
    dataDimensionType: 'DISAGGREGATION',
    skipTotal: false,
    categories: [
        { id: '<age-category-uid>' },
        { id: '<sex-category-uid>' },
    ],
};

const createCategoryComboMutation = {
    resource: 'categoryCombos',
    type: 'create' as const,
    data: categoryComboPayload,
};
```
