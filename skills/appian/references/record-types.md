# Record Types

## CLI Commands

```bash
# Create a record type
appian rt create --app $APP --file case.json

# List record types in an application
appian rt list --app $APP

# Get record type details (includes fields, source config, typeReference)
appian rt get <uuid>

# Update a record type
appian rt get $UUID | jq '.description = "Updated"' | appian rt update $UUID

# Delete a record type
appian rt delete <uuid>

# List fields
appian rt fields list <uuid>

# Add a field
echo '{"fieldName":"status","fieldType":"TEXT","length":50}' | appian rt fields add <uuid>

# List relationships
appian rt relationships list <uuid>

# Add a relationship
echo '{"relationshipName":"department","relationshipType":"MANY_TO_ONE","sourceRecordTypeFieldUuid":"<fk-field-uuid>","targetRecordTypeFieldUuid":"<pk-field-uuid>","targetRecordTypeUuid":"<target-rt-uuid>"}' | appian rt relationships add <rt-uuid>

# Insert record data (CSV via stdin or --file, PK column optional for auto-increment)
echo "value
Engineering
Finance
Sales" | appian records insert <rt-uuid>

# List record data
appian records list <rt-uuid>

# List views
appian rt views list <uuid>

# List actions
appian rt actions list <uuid>

# Add a record action (LIST_ACTION — appears on the record list)
echo '{"displayName":"Create Submission","processModelUuid":"<pm-uuid>","actionType":"LIST_ACTION","key":"createSubmission"}' | appian rt actions add <rt-uuid>

# Add a record action (RELATED_ACTION — appears on a record view row)
echo '{"displayName":"Edit Submission","processModelUuid":"<pm-uuid>","actionType":"RELATED_ACTION","key":"editSubmission","contextExpr":"=a!toJson(rv!record)","visibilityExpr":"=loggedInUser() = rv!record['"'"'recordType!{rt-uuid}Submission.fields.{fid}submittedByUsername'"'"']"}' | appian rt actions add <rt-uuid>

# Update an action
appian rt actions update <rt-uuid> <action-uuid> --file /tmp/action-update.json

# Delete an action
appian rt actions delete <rt-uuid> <action-uuid>

# List user filters
appian rt filters list <uuid>

# Add a user filter
echo '{"name":"Status","facetType":"LIST_OF_VALUES","sourceRef":"<field-uuid>"}' | appian rt filters add <rt-uuid>

# Update a user filter
echo '{"name":"Updated Name"}' | appian rt filters update <rt-uuid> <filter-uuid>

# Delete a user filter
appian rt filters delete <rt-uuid> <filter-uuid>
```

## Create JSON Schema

```json
{
  "name": "CM Case",
  "pluralName": "Cases",
  "sourceType": "DATABASE",
  "createTable": true,
  "description": "Support cases submitted by customers",
  "fields": [
    {"fieldName": "caseId", "fieldType": "INTEGER", "isPrimaryKey": true},
    {"fieldName": "title", "fieldType": "TEXT", "length": 255},
    {"fieldName": "description", "fieldType": "TEXT", "length": 4000},
    {"fieldName": "status", "fieldType": "TEXT", "length": 50},
    {"fieldName": "priority", "fieldType": "INTEGER"},
    {"fieldName": "assignedUsername", "fieldType": "USER"},
    {"fieldName": "customerId", "fieldType": "INTEGER"},
    {"fieldName": "createdByUsername", "fieldType": "USER"},
    {"fieldName": "modifiedByUsername", "fieldType": "USER"},
    {"fieldName": "createdAt", "fieldType": "DATETIME"},
    {"fieldName": "updatedAt", "fieldType": "DATETIME"}
  ]
}
```

## Source Configuration

- `sourceType`: `DATABASE` (most common), `WEB_SERVICE`, `PROCESS`, `SALESFORCE`
- `createTable`: `true` — Appian creates the DB table. `false` — table must already exist.
- `tableName`: UPPER_SNAKE_CASE, singular. Defaults to snake_case of record type name.
- `dataSourceUuid` and `schema`: for DATABASE types. Discover from existing record types in the same app if available.

## Field Configuration

Each field requires:
- `fieldName` — camelCase identifier used in SAIL expressions (e.g., `firstName`, `caseId`). The platform auto-generates the database column name in UPPER_SNAKE_CASE.
- `fieldType` — one of: TEXT, NUMBER, INTEGER, DECIMAL, DATE, DATETIME, TIME, BOOLEAN, USER, GROUP, DOCUMENT, FOLDER

Optional:
- `isPrimaryKey` — exactly one per record type, must be INTEGER
- `isUnique` — unique constraint
- `length` — for TEXT fields (0 = unlimited)
- `displayName`, `description`

Load `references/field-types.md` for the complete field type reference.

## Naming Conventions

- **Record type name**: `PREFIX EntityName` in Title Case, singular — `CM Case`, `CM Customer`, `CM Letter of Authorization`
- **Plural name**: Title Case, plural, no prefix — `Cases`, `Customers`, `Letters of Authorization`
- **Table name**: UPPER_SNAKE_CASE, singular — `CASE`, `CUSTOMER`, `LETTER_OF_AUTHORIZATION`
- **Field names**: camelCase — `caseId`, `createdAt`, `assignedUsername` (DB column auto-generated as UPPER_SNAKE_CASE)
- **Primary key**: `[entityName]Id` in camelCase — `caseId`, `employeeId`
- **Foreign keys**: `[referencedEntity]Id` in camelCase — `customerId`, `caseStatusId`

## Standard Field Structure

Order: primary key → business fields → foreign keys → audit fields.

1. Surrogate PK: `[table]_id` (INTEGER, isPrimaryKey: true)
2. Business fields from requirements
3. Foreign key fields (INTEGER)
4. Audit fields (entities only, not reference/junction tables):
   - `createdByUsername` (USER)
   - `modifiedByUsername` (USER)
   - `createdAt` (DATETIME)
   - `updatedAt` (DATETIME)

## Relationships

Relationships require both record types to exist first. Add them after creating all record types.

```bash
# Add a relationship (JSON via stdin)
echo '{
  "relationshipName": "customer",
  "sourceRecordTypeFieldUuid": "<fk-field-uuid>",
  "targetRecordTypeFieldUuid": "<pk-field-uuid>",
  "targetRecordTypeUuid": "<target-rt-uuid>",
  "relationshipType": "MANY_TO_ONE"
}' | appian rt relationships add <source-rt-uuid>
```

### Bidirectional Requirement

Always declare both sides:
- FK table: `MANY_TO_ONE` (or `ONE_TO_ONE` with unique FK)
- Referenced table: `ONE_TO_MANY` (or `ONE_TO_ONE`)

A one-sided relationship breaks record type traversal.

### Relationship Naming

- **MANY_TO_ONE**: remove `Id` suffix, keep camelCase — `customerId` → `customer`, `caseStatusId` → `caseStatus`
- **ONE_TO_MANY to entities**: pluralize target in camelCase — `CASE_NOTE` → `caseNotes`
- **ONE_TO_MANY to junction**: pluralize the other entity — CASE→CASE_TAG→TAG: name is `tags`
- **ONE_TO_ONE**: same as MANY_TO_ONE on FK side; entity name on other side

### Relationship Types

| Pattern | Implementation |
|---|---|
| One-to-Many | "many" holds FK (MANY_TO_ONE); "one" declares ONE_TO_MANY |
| Many-to-Many | Junction table with two FKs and two MANY_TO_ONE; both entities declare ONE_TO_MANY to junction |
| Self-Referential | Both directions on same RT (e.g., `parent_case_id` → parentCase / childCases) |

## Updating Record Types

**Always get before update** — updates replace provided fields entirely.

```bash
# Add a field to existing RT
echo '{"fieldName":"priority","fieldType":"INTEGER"}' | appian rt fields add $RT_UUID

# Update RT metadata
appian rt get $UUID | jq '.description = "New desc"' | appian rt update $UUID
```

## Record Actions

Record actions are process-model-backed operations surfaced in the record list (LIST_ACTION) or on individual record rows (RELATED_ACTION).

### JSON Schema

```json
{
  "displayName": "Edit Submission",
  "processModelUuid": "<pm-uuid>",
  "actionType": "RELATED_ACTION",
  "key": "editSubmission",
  "visibilityExpr": "=loggedInUser() = rv!record['recordType!{rt-uuid}Submission.fields.{fid}submittedByUsername']",
  "contextExpr": "={record: rv!record}",
  "icon": "f044",
  "dialogWidth": "MEDIUM_PLUS"
}
```

| Field | Required | Description |
|---|---|---|
| `displayName` | Yes | Label shown to users |
| `processModelUuid` | Yes | Process model to launch |
| `actionType` | Yes | `LIST_ACTION` (record list button) or `RELATED_ACTION` (per-row action) |
| `key` | Yes | Unique key for SAIL references (e.g., `recordType!Case.actions.editCase`) |
| `visibilityExpr` | No | SAIL expression → `true` shows the action, `false` hides it. Default: `=true()` (visible to all). Uses `rv!record` for field access. |
| `contextExpr` | No | SAIL expression passing record data to the process model. RELATED_ACTION only. Keys map to process parameter names. |
| `icon` | No | Font Awesome hex code (e.g., `f044` = pencil, `f1f8` = trash, `f067` = plus) |
| `dialogWidth` | No | `NARROW`, `MEDIUM`, `MEDIUM_PLUS` (default), `WIDE`, `EXTRA_WIDE`, `FIT` |
| `dialogHeight` | No | `AUTO` (default), `FIT`, `SHORT`, `MEDIUM`, `TALL`, `EXTRA_TALL` |

### Action Types

| Type | Where it appears | `rv!record` available | Typical use |
|---|---|---|---|
| `LIST_ACTION` | Button on the record list header | No | Create new records |
| `RELATED_ACTION` | Per-row action on record views/grids | Yes | Edit, delete, status transitions |

### Visibility Expressions

Control who sees the action. Omitting `visibilityExpr` or setting `"=true()"` makes the action visible to all users.

**Owner-only (record owner can edit):**
```
"visibilityExpr": "=loggedInUser() = rv!record['recordType!{rt-uuid}Entity.fields.{fid}createdByUsername']"
```

**Group-restricted (only admins):**
```
"visibilityExpr": "=a!isUserMemberOfGroup(loggedInUser(), cons!MY_ADMIN_GROUP)"
```

**Status-conditional (only when open):**
```
"visibilityExpr": "=rv!record['recordType!{rt-uuid}Case.fields.{fid}status'] = \"Open\""
```

**Combined (owner AND open status):**
```
"visibilityExpr": "=and(loggedInUser() = rv!record['recordType!{rt-uuid}Case.fields.{fid}assignedTo'], rv!record['recordType!{rt-uuid}Case.fields.{fid}status'] = \"Open\")"
```

### Context Expressions

For RELATED_ACTION, `contextExpr` passes the record into the process model. The keys in the dictionary must match process parameter names (case-sensitive):

```
"contextExpr": "={record: rv!record}"
```

The process model needs a parameter named `record` typed to the record type's `typeReference`.

### Examples

```bash
# LIST_ACTION — create new (no visibility restriction needed)
echo '{
  "displayName": "New Submission",
  "processModelUuid": "'$CREATE_PM_UUID'",
  "actionType": "LIST_ACTION",
  "key": "createSubmission",
  "icon": "f067"
}' | appian rt actions add $RT_UUID

# RELATED_ACTION — edit, restricted to record owner
echo '{
  "displayName": "Edit Submission",
  "processModelUuid": "'$UPDATE_PM_UUID'",
  "actionType": "RELATED_ACTION",
  "key": "editSubmission",
  "contextExpr": "={record: rv!record}",
  "visibilityExpr": "=loggedInUser() = rv!record['"'"'recordType!{rt-uuid}Submission.fields.{fid}submittedByUsername'"'"']",
  "icon": "f044"
}' | appian rt actions add $RT_UUID

# RELATED_ACTION — delete, restricted to admins
echo '{
  "displayName": "Delete",
  "processModelUuid": "'$DELETE_PM_UUID'",
  "actionType": "RELATED_ACTION",
  "key": "deleteSubmission",
  "visibilityExpr": "=a!isUserMemberOfGroup(loggedInUser(), cons!MY_ADMIN_GROUP)",
  "icon": "f1f8"
}' | appian rt actions add $RT_UUID
```

### Design Guidance

- **LIST_ACTION** (create new): typically no visibility restriction — any user who can see the record list can create
- **RELATED_ACTION** (edit/update): restrict to record owner via `visibilityExpr` comparing `loggedInUser()` to the owner/submitter field
- **RELATED_ACTION** (delete): restrict to admins or record owner
- Always set `contextExpr` on RELATED_ACTION to pass the record into the process model
- Use heredoc (`cat << 'EOF'`) for complex visibility expressions to avoid shell escaping issues
- The `key` must be unique within the record type and is used in SAIL references

## User Filters

User filters let end users filter the record list via dropdowns or date pickers. Three types: `LIST_OF_VALUES`, `DATE_RANGE`, `EXPRESSION`.

```bash
# Add a filter
echo '{"name":"Status","facetType":"LIST_OF_VALUES","sourceRef":"<field-uuid>"}' | appian rt filters add <rt-uuid>

# List existing filters
appian rt filters list <rt-uuid>
```

Add filters after record type fields and relationships exist (step 10 in dependency order). Load `references/record-type-user-filters.md` for full schemas, EXPRESSION filter examples, related record filtering, and design guidance.

## Common Pitfalls

- **Creating relationships before all RTs exist** — both source and target must exist first
- **One-sided relationships** — always declare both MANY_TO_ONE + ONE_TO_MANY
- **Composite primary keys** — Appian requires single-column INTEGER surrogate PK
- **Missing `createTable: true`** — without it, Appian expects table to already exist
- **Wrong field type names** — case-sensitive: TEXT, INTEGER, DATETIME (not text, integer)
- **Audit fields on junction/reference tables** — only entity tables get audit fields
- **Not discovering existing RTs** — always `appian rt list --app $APP` to check what exists and get dataSourceUuid/schema from existing RTs
