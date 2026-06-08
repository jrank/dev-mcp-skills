# Process Model Node Types

Complete reference for all supported node types, their activity class schema IDs, configuration shapes, and examples.

## Node Fields

Every node has these top-level fields:
- `id` — unique integer within the PM
- `type` — schema ID (see below)
- `name` — display label
- `coordinates` — `[x, y]` canvas position
- `connections` — list of target node IDs

Activity nodes (Script Task, User Input Task, Write Records, Send E-Mail) also support:
- `assignment` — attended/unattended config (required on activity nodes)
- `data` — inputs and outputs (schema-defined and custom)
- `forms` — interface config (for attended nodes only)

Gateway nodes support:
- `decision` — conditions and default path

## Tooled Node Types

---

### Start Event

- **Schema ID**: `core.0`
- **Description**: Entry point. Exactly one per process model.

```json
{
  "id": 1,
  "type": "core.0",
  "name": "Start",
  "coordinates": [50, 200],
  "connections": [2]
}
```

---

### End Event

- **Schema ID**: `core.1`
- **Description**: Termination point. Connections must be `[]`. Multiple allowed.

```json
{
  "id": 5,
  "type": "core.1",
  "name": "End",
  "coordinates": [850, 200],
  "connections": []
}
```

---

### XOR (Exclusive Gateway)

- **Schema ID**: `core.4`
- **Description**: Conditional branching. Evaluates conditions in order, routes to first match. Default path when none match.

Uses the `decision` field:

| Field | Type | Description |
|---|---|---|
| `decision.conditions[].expression` | string | SAIL boolean expression |
| `decision.conditions[].targetNodeId` | int | Node ID to route to when true |
| `decision.defaultPath` | int | Node ID when no condition matches |

```json
{
  "id": 3,
  "type": "core.4",
  "name": "Was Cancelled?",
  "coordinates": [450, 200],
  "connections": [4, 5],
  "decision": {
    "conditions": [
      {
        "expression": "=pv!cancel",
        "targetNodeId": 5
      }
    ],
    "defaultPath": 4
  }
}
```

---

### Script Task

- **Schema ID**: `internal.16`
- **Description**: Executes expressions and assigns results to process variables. Unattended.

Uses `assignment` + `data`:

```json
{
  "id": 2,
  "type": "internal.16",
  "name": "Set Defaults",
  "coordinates": [250, 200],
  "connections": [3],
  "assignment": {
    "attended": false
  },
  "data": {
    "outputs": [
      {
        "expression": "=\"Open\"",
        "saveInto": "pv!status"
      },
      {
        "expression": "=today()",
        "saveInto": "pv!createdDate"
      }
    ]
  }
}
```

---

### User Input Task

- **Schema ID**: `internal.17`
- **Description**: Presents a form to a user. Pauses until submit/cancel. Interface must exist first.

Uses `assignment` + `forms`:

```json
{
  "id": 2,
  "type": "internal.17",
  "name": "Employee Form",
  "coordinates": [250, 200],
  "connections": [3],
  "assignment": {
    "attended": true
  },
  "forms": {
    "interfaceUuid": "<interface-uuid>",
    "inputMap": {
      "record": "record",
      "isUpdate": "isUpdate",
      "cancel": "cancel"
    }
  }
}
```

`inputMap` keys = interface input names (without `ri!`), values = process variable names (without `pv!`).

---

### Write Records

- **Schema ID**: `internal3.write_records_to_source_23r3`
- **Description**: Writes a record to the data source. Creates if PK is null, updates if populated. Unattended.

Uses `assignment` + `data`. The specific input names for this node are schema-defined — check `appian pm nodes --help` to discover them. Typically you pass the record via `customInputs`:

```json
{
  "id": 4,
  "type": "internal3.write_records_to_source_23r3",
  "name": "Write Employee",
  "coordinates": [650, 200],
  "connections": [5],
  "assignment": {
    "attended": false
  },
  "data": {
    "customInputs": [
      {
        "name": "record",
        "expression": "pv!employee"
      }
    ]
  }
}
```

---

### Send E-Mail

- **Schema ID**: `internal3.sendemail3`
- **Description**: Sends an email. Unattended.

Uses `assignment` + `data`:

```json
{
  "id": 3,
  "type": "internal3.sendemail3",
  "name": "Send Confirmation",
  "coordinates": [450, 200],
  "connections": [4],
  "assignment": {
    "attended": false
  },
  "data": {
    "inputs": [
      {
        "name": "to",
        "expression": "pv!assignedUser"
      },
      {
        "name": "body",
        "expression": "=\"Your case has been created.\""
      }
    ]
  }
}
```

Note: Use `appian pm nodes --help` or check the node type schema for the exact input names available on Send E-Mail.

---

## Untooled Node Types

These can be included in process model definitions but require manual configuration in Appian Designer.

| Schema ID | Name | Capability |
|---|---|---|
| `internal3.rs2_ai_skill_document_classification3` | Classify Documents | AI document classification |
| `internal3.rs2_ai_skill_email_classification2` | Classify Emails | AI email classification |
| `internal3.rs2_ai_skill_document_extraction23r4` | Extract from Document | AI document extraction |
| `internal.database601` | Query Database | Direct SQL execution |
| `internal3.integration` | Call Integration | External API via connected system |

Include as placeholder nodes (no `data` or `assignment` needed):
```json
{
  "id": 3,
  "type": "internal3.integration",
  "name": "Call External API",
  "coordinates": [450, 200],
  "connections": [4]
}
```

---

## Complete Example: Form → Cancel Check → Write

```json
{
  "processVariables": [
    {"name": "employee", "type": "<typeReference from appian rt get>", "isParameter": true},
    {"name": "cancel", "type": "Boolean", "isParameter": true}
  ],
  "nodes": [
    {
      "id": 1, "type": "core.0", "name": "Start",
      "coordinates": [50, 200], "connections": [2]
    },
    {
      "id": 2, "type": "internal.17", "name": "Employee Form",
      "coordinates": [250, 200], "connections": [3],
      "assignment": {"attended": true},
      "forms": {
        "interfaceUuid": "<form-interface-uuid>",
        "inputMap": {"employee": "employee", "cancel": "cancel"}
      }
    },
    {
      "id": 3, "type": "core.4", "name": "Was Cancelled?",
      "coordinates": [450, 200], "connections": [4, 5],
      "decision": {
        "conditions": [{"expression": "=pv!cancel", "targetNodeId": 5}],
        "defaultPath": 4
      }
    },
    {
      "id": 4, "type": "internal3.write_records_to_source_23r3", "name": "Write Employee",
      "coordinates": [650, 200], "connections": [5],
      "assignment": {"attended": false},
      "data": {
        "customInputs": [{"name": "record", "expression": "pv!employee"}]
      }
    },
    {
      "id": 5, "type": "core.1", "name": "End",
      "coordinates": [850, 200], "connections": []
    }
  ]
}
```
