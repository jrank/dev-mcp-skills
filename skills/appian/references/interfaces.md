# Interfaces

## Create JSON Schema

```json
{
  "name": "CM_CaseForm",
  "description": "Create and update form for Case records",
  "inputs": [
    {"name": "record", "type": "record-type-reference-here"},
    {"name": "isUpdate", "type": "Boolean"},
    {"name": "cancel", "type": "Boolean"}
  ],
  "expression": "=a!formLayout(label: if(ri!isUpdate, \"Edit Case\", \"New Case\"), ...)"
}
```

## Naming Conventions

- **Prefix**: Application prefix + underscore (e.g., `CM_`)
- **Pattern**: `PREFIX_InterfacePurpose` in Title Case, no spaces
- **Examples**:
  - `CM_Dashboard` — main application dashboard
  - `CM_CaseSummary` — summary view for a Case record
  - `CM_CaseForm` — create/update form for Cases
  - `CM_CaseActionReassign` — record action form
  - `CM_CaseGrid` — reusable grid component

## Interface Types

| Type | Top-Level Layout | Purpose |
|---|---|---|
| Dashboard | `a!headerContentLayout()` | Landing page with KPIs, grids, navigation |
| Summary View | `a!headerContentLayout()` | Record detail display with actions |
| Form | `a!formLayout()` | Data entry for creating or updating records |
| Record Action Form | `a!formLayout()` | Form for record actions (reassign, close) |
| Wizard | `a!wizardLayout()` | Multi-step data entry |
| Reusable Component | No top-level layout | Fragment embedded in other interfaces |

## Input Patterns

### Standard Form Inputs

| Input | Type | Purpose |
|---|---|---|
| `record` | Record Type | The record data being created/edited |
| `isUpdate` | Boolean | `true` for edit, `false` for create |
| `cancel` | Boolean | Set by Cancel button for process flow control |

### Summary View Inputs

| Input | Type | Purpose |
|---|---|---|
| `record` | Record Type | The record to display |
| `identifier` | Integer | Record ID (alternative to full record) |

### Record Action Form Inputs

| Input | Type | Purpose |
|---|---|---|
| `record` | Record Type | The record being acted on |
| `cancel` | Boolean | Cancel flow control |

No `isUpdate` — record actions always operate on existing records.

### Dashboard Inputs

Usually none. Dashboards query their own data.

## Input Type References

For built-in types, use the friendly name: `Text`, `Number (Integer)`, `Number (Decimal)`, `Boolean`, `Date`, `Date and Time`, `Time`, `User`, `Group`, `Document`, `Folder`, `Encrypted Text`, `Map`

For record type inputs, use the `typeReference` string from the record type's details (get the record type to find it).

## Operation Sequence

**Creating with inputs and expression:**
1. Provide `inputs` array and `expression` together in the create call
2. The expression can reference inputs via `ri!` prefix matching input names

**Updating an existing interface:**
1. Get the interface to see current state
2. Modify inputs or expression as needed
3. Update with the modified payload

If changing inputs, ensure the expression still references correct `ri!` names.

## UX Patterns

### Dashboard
1. Header — app name, subtitle
2. KPI row — 3-5 metric cards in `a!columnsLayout()`
3. Primary grid — main work queue, filterable
4. Quick actions — links to common workflows

### Summary View
1. Header — record title, status badge, key IDs
2. Detail sections — grouped fields by business domain
3. Related data — grids of child/related records
4. Action buttons — edit, reassign, close

### Form
1. Title — dynamic: "New [Entity]" or "Edit [Entity]"
2. Field sections — grouped by business meaning
3. Buttons — Submit (primary) + Cancel (secondary)

### Record Action Form
1. Context display — key record fields as read-only
2. Action fields — only what's needed for the action
3. Buttons — action-specific label ("Reassign", "Close Case") + Cancel

## Conventions

- One interface per purpose (don't combine unrelated views)
- Create separate interfaces for: dashboard, each summary view, each form, each record action
- Forms with Cancel need the cancel button to save `true` into `ri!cancel`
- Required fields before optional within sections
- Group fields by business meaning, not by data type

## Common Pitfalls

- **Setting expression before inputs** — expression fails if it references `ri!` variables that don't exist yet; provide both together or add inputs first
- **Mismatched `ri!` names** — expression references must exactly match input names
- **Wrong top-level layout** — `a!formLayout()` for forms, `a!headerContentLayout()` for dashboards/summaries
- **Plain-text record type names in expressions** — use UUID-qualified format: `'recordType!{uuid}Name.fields.{fieldUuid}fieldName'`
- **Forgetting Cancel button on forms** — process model forms need Cancel to save into `ri!cancel`
- **Overloading a single interface** — keep dashboard, summary, and form separate

## SAIL Guidance

For SAIL syntax, components, layouts, and interaction patterns, load `references/sail.md`.
