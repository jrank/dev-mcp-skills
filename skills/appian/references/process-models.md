# Process Models

## CLI Commands

```bash
# Create a process model
appian pm create --app $APP --file create-employee-pm.json

# List process models
appian pm list --app $APP

# Get process model details (includes nodes, variables, start form)
appian pm get <uuid>

# Update a process model (nodes, variables, connections, start form)
appian pm update <uuid> --file updated-pm.json

# Delete
appian pm delete <uuid>

# Run a process model (unattended only) and get results
echo '{"inputs":{"caseRecord":...}}' | appian pm run <uuid>

# List nodes in a process model
appian pm nodes list <uuid>
```

Note: Creating a process model requires `--app $APP` (to associate with the application), `parentFolderUuid` (a PM folder, not a regular folder), and `errorAlertGroupName`. Discover the PM folder from `appian apps get $APP` (look at `defaultObjects.processModelFolderUuid`). Use the application's administrators group for `errorAlertGroupName`.

## Update JSON Schema

```json
{
  "description": "Creates a new case record from the submitted form",
  "processVariables": [
    {"name": "record", "type": "<typeReference from appian rt get>", "isParameter": true, "isRequired": false},
    {"name": "isUpdate", "type": "BOOLEAN", "isParameter": true},
    {"name": "cancel", "type": "BOOLEAN"}
  ],
  "nodes": [
    {"id": 1, "type": "core.0", "name": "Start", "coordinates": [50, 200], "connections": [2]},
    {"id": 2, "type": "internal.17", "name": "Case Form", "coordinates": [250, 200], "connections": [3]},
    {"id": 3, "type": "core.4", "name": "Was Cancelled?", "coordinates": [450, 200], "connections": [4, 5]},
    {"id": 4, "type": "internal3.write_records_to_source_23r3", "name": "Write Case", "coordinates": [650, 100], "connections": [5]},
    {"id": 5, "type": "core.1", "name": "End", "coordinates": [850, 200], "connections": []}
  ],
  "startForm": {
    "interfaceUuid": "<interface-uuid>",
    "inputMap": {"record": "record", "isUpdate": "isUpdate", "cancel": "cancel"}
  }
}
```

## Naming Conventions

- **Prefix**: Application prefix + space (e.g., `CM `)
- **Pattern**: `PREFIX ActionName` in Title Case
- **Name describes the business action**
- **Examples**:
  - `CM Create Case`
  - `CM Reassign Case`
  - `CM Close Case`
  - `CM Send Notification`

## Required Configuration

Every process model needs:
- `name` — follows naming conventions
- `description` — what it does and when it runs
- `parentFolderUuid` — a process model folder (not a regular folder)
- `securityGroupName` — group controlling access (typically PREFIX Administrators)

## Nodes and Flow

### Node Structure

Each node:
- `id` — unique integer (start at 1, increment)
- `type` — schema ID (see node types below)
- `coordinates` — `[x, y]` canvas position
- `connections` — list of target node IDs (empty for End Event)
- `name` — display label
- `data` — type-specific configuration (inputs, outputs)
- `assignment` — for activity nodes (attended vs unattended)
- `forms` — for attended nodes (interface reference)
- `decision` — for XOR gateways (conditions)

### Core Node Types

| Schema ID | Name | Purpose |
|---|---|---|
| `core.0` | Start Event | Entry point (exactly one) |
| `core.1` | End Event | Termination (connections must be `[]`) |
| `core.4` | XOR Gateway | Conditional branching |
| `internal.16` | Script Task | Execute expressions, assign variables |
| `internal.17` | User Input Task | Present form, collect input |
| `internal3.write_records_to_source_23r3` | Write Records | Write record to data source |
| `internal3.sendemail3` | Send E-Mail | Send email |

Load `references/node-types.md` for the complete reference with all schema IDs and configuration shapes.

### Flow Patterns

**Linear** (Start → Task → End):
```
Start(1) → Script(2) → End(3)
```

**Form submission** (Start → Form → Write → End):
```
Start(1) → UserInput(2) → WriteRecords(3) → End(4)
```

**Form with cancel** (Start → Form → XOR → Write/Skip → End):
```
Start(1) → UserInput(2) → XOR(3) → WriteRecords(4) → End(5)
                                   → End(5)  [cancel path]
```

### Canvas Layout

- Left to right for main flow
- Start at `[50, 200]`
- ~200px horizontal spacing
- XOR branches: offset vertically (upper y=100, lower y=300)

## Process Variables

Each variable:
- `name` — camelCase (`caseRecord`, `isApproved`)
- `type` — built-in: Text, Number (Integer), Number (Decimal), Boolean, Date, Time, Date and Time, User, Group, Document, Folder. For record types: use the `typeReference` string from `appian rt get <uuid>`
- `isParameter` (optional) — `true` = visible to callers
- `isRequired` (optional) — callers must provide

- `multiple` (optional) — holds a list

### Common Variable Patterns

**Record create/update:**
- `record` (type: typeReference from `appian rt get`, isParameter: true)
- `isUpdate` (BOOLEAN, isParameter: true)
- `cancel` (BOOLEAN)

**Notification:**
- `recipientUser` (USER, isParameter: true, isRequired: true)
- `messageBody` (TEXT, isParameter: true)
- `subject` (TEXT, isParameter: true)

## Start Forms

Connect an interface as a start form using `startForm`:

```json
{
  "startForm": {
    "interfaceUuid": "<uuid>",
    "inputMap": {
      "record": "record",
      "isUpdate": "isUpdate",
      "cancel": "cancel"
    }
  }
}
```

`inputMap` keys = process variable names (without `pv!`), values = interface input names (without `ri!`).

## Security

- `securityGroupName` required — use the application's administrators group
- Process models live in PM folders only (not regular folders)
- Discover PM folders: look at app structure after creation

## Updating Process Models

**Always get before update** — node/variable lists replace entirely.

```bash
# Get current state
appian pm get $UUID > current.json

# Modify and update
cat current.json | jq '.nodes += [{"id":6,"type":"internal.16","name":"Log","coordinates":[300,200],"connections":[5]}]' | appian pm update $UUID
```

## Common Pitfalls

- **PM in regular folder** — must be in a process model folder
- **Duplicate node IDs** — each must be unique within the PM
- **End Event with connections** — must have `connections: []`
- **Missing Start Event** — every PM needs exactly one `core.0`
- **XOR without conditions** — needs `decision.conditions` for branching logic
- **Mismatched interface input names in start form** — `inputMap` values must match interface inputs exactly
- **Updating without existing nodes** — replaces entire node list; include all existing nodes
- **Interface must exist before referencing** — create interface before PM references it
- **Forgetting cancel handling** — forms with Cancel need XOR after User Input to check cancel variable
- **Plain-text record type names in expressions** — use UUID-qualified format
