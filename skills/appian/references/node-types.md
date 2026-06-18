# Process Model Node Types

## Node Fields

Every node has these top-level fields:
- `id` — unique integer within the PM
- `type` — schema ID
- `name` — display label
- `coordinates` — `[x, y]` canvas position
- `connections` — list of target node IDs

Activity nodes also support:
- `assignment` — attended/unattended config (required on activity nodes)
- `data` — inputs and outputs (schema-defined and custom)
- `forms` — interface config (for attended nodes only)

Gateway nodes support:
- `decision` — conditions and default path

## Discovering Node Type Schemas

Use the node type discovery tools to list all available node types and get the input/output schema for a specific type. When working with attended nodes, provide the form interface UUID to discover form inputs. When working with integrations or subprocesses, provide the reference UUID for schema enrichment.

Use this to discover the exact input names and required fields before configuring unfamiliar node types.

## Structural Patterns

Four distinct patterns for how nodes are configured:

### Gateway pattern: `decision`

XOR (`core.4`) uses the `decision` field — not discoverable via schema:

```json
{
  "id": 3, "type": "core.4", "name": "Was Cancelled?",
  "coordinates": [450, 200], "connections": [4, 5],
  "decision": {
    "conditions": [
      {"expression": "=pv!cancel", "targetNodeId": 5}
    ],
    "defaultPath": 4
  }
}
```

### Unattended activity pattern: `assignment` + `data.outputs`

Script Task (`internal.16`) — schema returns empty inputs/outputs, use `data.outputs` with `expression`/`saveInto`:

```json
{
  "id": 2, "type": "internal.16", "name": "Set Defaults",
  "coordinates": [250, 200], "connections": [3],
  "assignment": {"attended": false},
  "data": {
    "outputs": [
      {"expression": "=\"Open\"", "saveInto": "pv!status"},
      {"expression": "=today()", "saveInto": "pv!createdDate"}
    ]
  }
}
```

### Attended activity pattern: `assignment` + `forms`

User Input Task (`internal.17`) — uses `forms` with interface reference:

```json
{
  "id": 2, "type": "internal.17", "name": "Employee Form",
  "coordinates": [250, 200], "connections": [3],
  "assignment": {"attended": true},
  "forms": {
    "interfaceUuid": "<interface-uuid>",
    "inputMap": {"employee": "employee", "cancel": "cancel"}
  }
}
```

`inputMap` keys = interface input names (without `ri!`), values = process variable names (without `pv!`).

### Schema-defined inputs pattern: `assignment` + `data.inputs`

Write Records (`internal3.write_records_to_source_23r3`) — inputs discovered via the node type schema:

```json
{
  "id": 4, "type": "internal3.write_records_to_source_23r3", "name": "Write Employee",
  "coordinates": [650, 200], "connections": [5],
  "assignment": {"attended": false},
  "data": {
    "inputs": [
      {"name": "Records", "expression": "={pv!employee}"},
      {"name": "Version", "value": 6},
      {"name": "CaptureEvents", "value": false}
    ]
  }
}
```

Note: `Records` wraps in curly braces to create a list. Discover input names with the node type schema tool.

## Common Schema IDs

| Schema ID | Name |
|---|---|
| `core.0` | Start Event |
| `core.1` | End Event (connections must be `[]`) |
| `core.4` | XOR (Exclusive Gateway) |
| `internal.16` | Script Task |
| `internal.17` | User Input Task |
| `internal3.write_records_to_source_23r3` | Write Records |
| `internal3.sendemail3` | Send E-Mail |
| `internal3.integration` | Call Integration |

Use the node type listing tool for the full catalog.

## Complete Example: Form → Cancel Check → Write

```json
{
  "processVariables": [
    {"name": "employee", "type": "<typeReference from record type details>", "isParameter": true},
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
        "inputs": [
          {"name": "Records", "expression": "={pv!employee}"},
          {"name": "Version", "value": 6},
          {"name": "CaptureEvents", "value": false}
        ]
      }
    },
    {
      "id": 5, "type": "core.1", "name": "End",
      "coordinates": [850, 200], "connections": []
    }
  ]
}
```
