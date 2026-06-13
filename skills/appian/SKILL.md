---
name: "appian"
description: "Build and modify Appian applications using the Appian CLI. Covers applications, record types, interfaces, expression rules, process models, sites, Web APIs, constants, groups, folders, and documents. Use this skill whenever working with ANY Appian platform object — including data modeling, relationships, SAIL expressions, forms, dashboards, workflows, deployments, or application structure. Activate for any task that creates, reads, updates, or deletes Appian design objects."
---

## The Appian CLI

All Appian platform operations use the `appian` CLI bundled with this skill at `./scripts/appian`. Always `cd` into this skill's directory before running commands so relative paths resolve correctly.

### Command Structure

```
appian <resource> <action> [uuid] [flags]
```

Resources and their aliases:
- `applications` (alias: `apps`)
- `record-types` (alias: `rt`)
- `expression-rules` (alias: `er`)
- `interfaces`
- `process-models` (alias: `pm`)
- `sites`

Actions common to most resources: `list`, `get`, `create`, `update`, `delete`

### Global Flags

| Flag | Purpose |
|---|---|
| `--app <uuid>` | Scope operations to an application |
| `--env <name>` | Target a specific environment (default from config) |
| `--file <path>` | Read JSON request body from a file |
| `--format <fmt>` | Output format: `json` (default), `table`, `csv`, `uuids` |
| `--quiet` | Suppress non-essential output |
| `--verbose` | Show request/response details for debugging |

### Input: JSON via stdin or --file

Create and update commands accept JSON bodies. Two ways to provide them:

**Pipe from stdin:**
```bash
echo '{"name":"MyRecord","sourceType":"DATABASE","fields":[...]}' | appian rt create --app $APP
```

**Use --file for complex payloads (write to /tmp, not the skill directory):**
```bash
appian rt create --app $APP --file /tmp/record-type.json
```

**Pipe get → modify → update (idiomatic pattern):**
```bash
appian rt get $UUID | jq '.description = "Updated description"' | appian rt update $UUID
```

**Use heredoc for SAIL expressions (avoids quote/brace escaping hell):**
```bash
cat << 'EOF' | appian interfaces create --app $APP
{
  "name": "MY_EmployeesPage",
  "expression": "=a!gridField(label: \"Employees\", data: recordType!{uuid}Employee, columns: {a!gridColumn(label: \"Name\", value: fv!row['recordType!{uuid}Employee.fields.{fid}firstName'])})"
}
EOF
```

```bash
cat << 'EOF' | appian pm create --app $APP
{
  "name": "MY Create Case",
  "processVariables": [{"name": "record", "type": "{urn:com:appian:recordtype:datatype}abc-123", "isParameter": true}],
  "startForm": {"interfaceUuid": "uuid-here", "inputMap": {"record": "record"}},
  "nodes": [
    {"id": 1, "type": "core.0", "name": "Start", "coordinates": [50, 200], "connections": [2]},
    {"id": 2, "type": "internal3.write_records_to_source_23r3", "name": "Write", "coordinates": [250, 200], "connections": [3]},
    {"id": 3, "type": "core.1", "name": "End", "coordinates": [450, 200], "connections": []}
  ]
}
EOF
```

### Output

All commands return JSON by default. Use `--format uuids` when you only need the UUID of a created object. Use `jq` to extract specific fields:

```bash
APP=$(echo '{"name":"My App"}' | appian apps create --format uuids)
appian rt list --app $APP | jq '.[].name'
```

### Error Handling

The CLI returns non-zero exit codes on failure with JSON error details on stderr. Check exit codes after critical operations:

```bash
appian rt create --app $APP --file body.json || echo "Create failed"
```

### Authentication

Auth is configured in `~/.appian/config.yaml`. The CLI handles credentials automatically — no auth flags needed per-command. Use `appian status` to verify connectivity.

---

## Resource Reference Map

Each resource has a dedicated reference file with full CLI examples, JSON schemas, design conventions, and pitfalls. Load the relevant reference for your task:

| When to load | Reference File |
|---|---|
| Creating or managing an application | `references/applications.md` |
| Creating/modifying record types, fields, relationships, views, or actions | `references/record-types.md` |
| Requirements mention filtering, searching, faceted navigation, or record list dropdowns | `references/record-type-user-filters.md` |
| Creating/modifying interfaces or writing SAIL form expressions | `references/interfaces.md` |
| Creating/modifying expression rules | `references/expression-rules.md` |
| Creating/modifying process models, adding nodes, or wiring start forms | `references/process-models.md` |
| Creating/modifying sites or adding pages | `references/sites.md` |
| Creating constants, groups, folders, or documents | `references/supporting-objects.md` |
| Designing a data model, choosing entity structure, or normalizing fields into lookup tables | `references/data-modeling.md` |
| Writing SAIL expressions for interfaces (layout, components, patterns) | `references/sail.md` |
| Using Appian functions, operators, or type conversions in expressions | `references/expressions.md` |
| Configuring security roles, record-level security, or group hierarchy | `references/security.md` |
| Starting a multi-object task — need to plan dependency order and scope | `references/change-planning.md` |
| Validating or testing completed changes | `references/change-review.md` |
| Choosing field types or need type constraints (length, precision) | `references/field-types.md` |
| Configuring process model nodes (smart services, gateways, events) | `references/node-types.md` |
| Configuring record events, writing events in process models, displaying event history in interfaces, or enabling process mining (Process HQ). Requirements mention: auditing, activity log, event history, tracking changes, collaboration on records, process mining. | `references/record-events.md` |
| Building a dashboard, form layout, or summary view | `references/ui-patterns.md` |
| Need to look up a specific SAIL component's parameters | `references/component-reference.md` |

### Loading Strategy

For a typical task:
1. Load the primary resource reference (e.g., `references/record-types.md` for record type work)
2. Load supplementary references as needed (e.g., `references/data-modeling.md` for schema design, `references/field-types.md` for type constraints)
3. Load `references/sail.md` when writing SAIL expressions for interfaces
4. Load `references/change-planning.md` when starting a multi-object task to understand dependency ordering

---

## Dependency Order

Appian objects must be created in dependency order. Later objects reference earlier ones:

1. Application (creates default groups and folders)
2. Groups (additional role groups)
3. Folders (additional sub-folders)
4. Constants (reference groups, store config values)
5. Record types (with fields)
6. Record type relationships (requires all record types to exist)
7. Expression rules
8. Interfaces
9. Process models
10. Record type actions, views, filters (reference process models and interfaces)
11. Sites (reference interfaces)
12. Web APIs
13. Documents (uploaded to folders)

---

## Common Workflow

A typical application build follows this sequence:

```bash
# 1. Create application
APP=$(echo '{"name":"Case Management"}' | appian apps create --format uuids)

# 2. Discover defaults (auto-generated groups, folders)
appian rt list --app $APP   # empty initially, but shows app is ready

# 3. Create record types
appian rt create --app $APP --file case-record-type.json

# 4. Create interfaces
appian interfaces create --app $APP --file case-form.json

# 5. Create process models
appian pm update $PM_UUID --file create-case-pm.json

# 6. Wire it together (record actions, sites)
appian sites create --app $APP --file case-site.json
```

---

## Tips

- Always discover what exists before creating (`appian rt list --app $APP`)
- Get an object before updating it — updates replace provided fields entirely
- Use `--format uuids` when piping creation output into subsequent commands
- Store UUIDs in variables for multi-step workflows
- Record type relationships require both sides declared (MANY_TO_ONE + ONE_TO_MANY)
- SAIL expressions in JSON: use heredoc (`cat << 'EOF'`) to avoid quote escaping issues
- All `recordType!` references in expressions must use UUID-qualified format: `'recordType!{uuid}Name.fields.{fieldUuid}fieldName'`
