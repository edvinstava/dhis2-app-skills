# Sharing metadata

DHIS2 controls access to metadata objects through a sharing model. Every shareable object
carries a `publicAccess` string, per-user grants, per-group grants, and an `externalAccess`
flag. A dedicated endpoint, `/api/sharing`, provides a clean read/write surface for these
settings without requiring a full object update.

Before calling any sharing endpoint, read the controller source for the version you're
targeting (see [../data-fetching.md](../data-fetching.md) step 1). The source is at
`dhis-2/dhis-web-api/src/main/java/org/hisp/dhis/webapi/controller/SharingController.java`.

---

## The sharing model

Every shareable object carries these fields:

| Field | Type | Meaning |
|-------|------|---------|
| `publicAccess` | 8-char string | Access granted to all authenticated users |
| `externalAccess` | boolean | Whether anonymous (unauthenticated) access is allowed |
| `userAccesses` | array | Per-user grants: `{ id, name, displayName, access }` |
| `userGroupAccesses` | array | Per-group grants: `{ id, name, displayName, access }` |
| `createdBy` / `user` | reference | Owner — the user who created the object (`user` is an alias derived from `createdBy`) |

### Two sharing shapes in the API

The `sharing` block appears in two different shapes depending on where you read it:

**Full-object response** (`GET /api/dataElements/<uid>`) — the inline `sharing` field uses
a map format with short key names:

```json
{
  "sharing": {
    "owner": "GOLswS44mh8",
    "external": false,
    "public": "rw------",
    "users": {},
    "userGroups": {}
  }
}
```

**`/api/sharing` response** — uses a list format with expanded key names, wrapped in
`object` and `meta` envelopes:

```json
{
  "meta": { "allowPublicAccess": true, "allowExternalAccess": false },
  "object": {
    "id": "FTRrcoaog83",
    "displayName": "Accute Flaccid Paralysis (Deaths < 5 yrs)",
    "publicAccess": "rw------",
    "externalAccess": false,
    "user": { "id": "GOLswS44mh8", "name": "Tom Wakiki" },
    "userAccesses": [],
    "userGroupAccesses": []
  }
}
```

The `meta` block tells you whether the current user can set public or external access on
this object. Always read from `/api/sharing` when you need the sharing state for UI display
— the format is stable and includes the `meta` hints.

---

## The access string

DHIS2 represents access as an 8-character string. Positions encode read/write flags:

| Positions | Meaning | Notes |
|-----------|---------|-------|
| 1–2 | Metadata read/write | Applies to all shareable types |
| 3–4 | Data read/write | Only meaningful when the type has `dataShareable: true` (e.g. `dataSet`, `program`, `categoryOption`) |
| 5–8 | Reserved | Always `----` in 2.42 — `AccessStringHelper` validates that the string ends with `"----"` |

Validation rule from `AccessStringHelper.java` (confirmed in 2.42 source):

```
length must be exactly 8
string must end with "----"
byte[0]: 'r' or '-'   (position 1 in the table above — metadata read)
byte[1]: 'w' or '-'   (position 2 — metadata write)
byte[2]: 'r' or '-'   (position 3 — data read)
byte[3]: 'w' or '-'   (position 4 — data write)
```

Note: byte indices are 0-based; positions 1–4 in the table above correspond to byte
indices 0–3.

The JavaDoc comment at the top of `AccessStringHelper.java` says "only the two first
positions are used". That comment is outdated — positions 3–4 (`DATA_READ`/`DATA_WRITE`)
are actively enforced for `dataShareable` types. For non-`dataShareable` types (e.g.
`dataElement`, `indicator`, `dashboard`), the server silently strips data-sharing bits
on write — safe to send `rw------` to a non-`dataShareable` type; sending `rwrw----`
for the same type would have the data bits silently stripped to `rw------`.

**Common combinations:**

| String | Meaning |
|--------|---------|
| `rwrw----` | Full metadata + data access |
| `rwr-----` | Metadata read/write + data read |
| `rw------` | Edit metadata; no data access (or type is not `dataShareable`) |
| `r-------` | Read metadata only |
| `--------` | No access |

**Named constants from `AccessStringHelper.java`:**

| Constant | Value |
|----------|-------|
| `DEFAULT` | `--------` |
| `READ` | `r-------` |
| `WRITE` | `-w------` | Metadata write only |
| `READ_WRITE` | `rw------` |
| `DATA_READ` | `--r-----` |
| `DATA_WRITE` | `---w----` |
| `DATA_READ_WRITE` | `--rw----` |
| `FULL` | `rwrw----` |
| `CATEGORY_OPTION_DEFAULT` | `rwrw----` (default CategoryOption only) |
| `CATEGORY_NO_DATA_SHARING_DEFAULT` | `rw------` (default Category/CategoryCombo) |

---

## Reading sharing — GET /api/sharing

```
GET /api/sharing?type=<singular>&id=<uid>
```

The `type` parameter uses the **singular schema name** (e.g. `dataElement`, not
`dataElements`). Find the singular name at `/api/schemas` or from the `singular` field in
`/api/schemas/<type>` (see [./schemas.md](./schemas.md)).

**Example:**

```
GET /api/sharing?type=dataElement&id=FTRrcoaog83
```

Response (confirmed against live 2.42 play instance):

```json
{
  "meta": {
    "allowPublicAccess": true,
    "allowExternalAccess": false
  },
  "object": {
    "id": "FTRrcoaog83",
    "name": "Accute Flaccid Paralysis (Deaths < 5 yrs)",
    "displayName": "Accute Flaccid Paralysis (Deaths < 5 yrs)",
    "publicAccess": "rw------",
    "externalAccess": false,
    "user": { "id": "GOLswS44mh8", "name": "Tom Wakiki" },
    "userAccesses": [],
    "userGroupAccesses": []
  }
}
```

- `meta.allowPublicAccess` — whether the current user may change `publicAccess` on this object.
- `meta.allowExternalAccess` — whether the current user may toggle `externalAccess`.
- `object.user` — the owner (derived from `createdBy`); read-only here.
- `object.userGroupAccesses` items have shape `{ id, name, displayName, access }` — prefer `displayName` for display; `name` is also present but carries the same value.
- `object.userAccesses` items have shape `{ id, name, displayName, access }` — same note applies.

---

## Writing sharing

The controller accepts both `POST` and `PUT` — `@PutMapping` in `SharingController.java`
delegates to the same `postSharing` handler. All patterns below use `type: 'update' as const`,
which `@dhis2/app-runtime` sends as PUT. That works because the endpoint identifies the
target via `params: { type, id }` rather than a path-segment id.

**Request shape:**

```
POST /api/sharing?type=<singular>&id=<uid>
Content-Type: application/json

{
  "object": {
    "publicAccess": "rw------",
    "externalAccess": false,
    "userAccesses": [
      { "id": "<user-uid>", "access": "rw------" }
    ],
    "userGroupAccesses": [
      { "id": "<group-uid>", "access": "r-------" }
    ]
  }
}
```

The body must be wrapped in `{ "object": { ... } }` — the controller deserializes the
request as a `Sharing` object and reads from `sharing.getObject()`. Sending a flat payload
without the `object` envelope will result in a null-access error.

The server validates every access string via `AccessStringHelper.isValid()`. Invalid
strings (wrong length, wrong tail, invalid chars) return HTTP 409 Conflict.

For non-`dataShareable` types, the server automatically strips data-sharing bits from any
access strings that include them — you will not get an error, but the data bits will be
silently removed.

---

## What's shareable

Not all types support sharing. The authoritative list is at:

```
GET /api/schemas.json?fields=singular,shareable,dataShareable
```

Filter on `shareable: true` to get the types that accept sharing. Filter additionally on
`dataShareable: true` to find the subset where positions 3–4 of the access string are
meaningful.

Types where `dataShareable: true` in 2.42 (confirmed from schemas snapshot):
`categoryOption`, `dataSet`, `trackedEntityType`, `programStage`, `program`,
`relationshipType`, `sqlView`, `aggregateDataExchange`. For all other shareable types,
only positions 1–2 of the access string have effect.

**Organisation units are not shareable** (`shareable: false` on `organisationUnit` in
2.42). Access control for org units is managed via the hierarchy assigned to users, not
object-level sharing. See [./common-types.md](./common-types.md) for details.

The `/api/sharing` endpoint itself will return HTTP 409 if called with a non-shareable
type: `"Type <type> is not supported."`.

---

## Common patterns

All patterns use `useDataEngine` from `@dhis2/app-runtime`. See
[../data-fetching.md](../data-fetching.md) for the full mutation hook pattern (error
handling, cache invalidation, alerts).

### Make an object public read-only

```typescript
import { useDataEngine } from '@dhis2/app-runtime';

const dataEngine = useDataEngine();

const makePublicReadOnly = {
    resource: 'sharing',
    type: 'update' as const,
    params: { type: 'dataElement', id: 'FTRrcoaog83' },
    data: {
        object: {
            publicAccess: 'r-------',
            externalAccess: false,
            userAccesses: [],
            userGroupAccesses: [],
        },
    },
};

await dataEngine.mutate(makePublicReadOnly);
```

### Share with a user group (read/write)

```typescript
const shareWithGroup = {
    resource: 'sharing',
    type: 'update' as const,
    params: { type: 'dataElement', id },
    data: {
        object: {
            publicAccess: '--------',
            externalAccess: false,
            userAccesses: [],
            userGroupAccesses: [
                { id: groupId, access: 'rw------' },
            ],
        },
    },
};

await dataEngine.mutate(shareWithGroup);
```

### Remove all user and group accesses

```typescript
const removeAllAccesses = {
    resource: 'sharing',
    type: 'update' as const,
    params: { type: 'dataElement', id },
    data: {
        object: {
            publicAccess: '--------',
            externalAccess: false,
            userAccesses: [],
            userGroupAccesses: [],
        },
    },
};

await dataEngine.mutate(removeAllAccesses);
```

### Copy sharing from another object

Fetch the source object's sharing, then write it to the target. Use the `/api/sharing`
GET response shape directly as the POST body — it is already in `{ object: { ... } }` form.
The `user` (owner) field in the copied object is ignored by the controller — ownership is
read from the target record, not from the request body.

```typescript
const copySharingFromSource = async (
    type: string,
    sourceId: string,
    targetId: string,
) => {
    // GET returns { meta, object } — reuse object as-is
    const source = await dataEngine.query({
        sharing: {
            resource: 'sharing',
            params: { type, id: sourceId },
        },
    });

    await dataEngine.mutate({
        resource: 'sharing',
        type: 'update' as const,
        params: { type, id: targetId },
        data: { object: source.sharing.object },
    });
};
```

Note: `externalAccess` in the copied `object` will be honoured only if the current user
has `allowExternalAccess` permission on the target. If not, the server silently ignores
that field.
