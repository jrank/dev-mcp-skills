# Appian CLI Tool Reference

## Discovery

The CLI has built-in help at every level:
```
appian --help                        # list all resources
appian <resource> --help             # list actions for a resource
appian <resource> <action> --help    # show flags, JSON schema, examples
```

Use these to discover available commands and their parameters. This reference covers patterns and conventions that help isn't great at communicating.

## Setup

The `appian` CLI is bundled with the skill at `./scripts/appian`. Always `cd` into the skill's directory before running commands so relative paths resolve correctly.

Auth is configured in `~/.appian/config.yaml`. The CLI handles credentials automatically — no auth flags needed per-command. Use `appian status` to verify connectivity.

## Command Structure

```
appian <resource> <action> [uuid] [flags]
```

Common resource aliases:
- `applications` → `apps`
- `record-types` → `rt`
- `expression-rules` → `er`
- `process-models` → `pm`
- `documents` → `docs`

## Global Flags

| Flag | Purpose |
|---|---|
| `--app <uuid>` | Scope operations to an application |
| `--env <name>` | Target a specific environment (default from config) |
| `--file <path>` | Read JSON request body from a file |
| `--format <fmt>` | Output format: `json` (default), `table`, `csv`, `uuids` |
| `--quiet` | Suppress non-essential output |
| `--verbose` | Show request/response details for debugging |

## Input Patterns

All create/update commands accept JSON. Three ways to provide it:

**Pipe from stdin (simple payloads):**
```bash
echo '{"name":"My App","prefix":"MA"}' | appian apps create
```

**Heredoc (complex payloads with SAIL expressions — avoids quote escaping):**
```bash
cat << 'EOF' | appian interfaces create --app $APP
{
  "name": "MA_Dashboard",
  "expression": "=a!gridField(...)"
}
EOF
```

**File (reusable or very large payloads — write to /tmp, not the skill directory):**
```bash
appian rt create --app $APP --file /tmp/record-type.json
```

## Output & UUID Capture

All commands return JSON by default. Use `--format uuids` to capture just the UUID:
```bash
APP=$(echo '{"name":"My App"}' | appian apps create --format uuids)
```

Extract specific fields with `jq`:
```bash
appian rt list --app $APP | jq '.[].name'
```

Pipe get → modify → update:
```bash
appian rt get $UUID | jq '.description = "Updated"' | appian rt update $UUID
```

## Error Handling

Non-zero exit codes on failure. JSON error details on stderr.
```bash
appian rt create --app $APP --file body.json || echo "Create failed"
```

## Key Patterns

- Always list before creating to discover what exists
- Store UUIDs in shell variables for multi-step workflows
- Use `--format uuids` when piping creation output into subsequent commands
- Get an object before updating — updates replace all provided fields
- CSV format for record data operations (pipe via stdin or `--file`)
- Use heredoc (`cat << 'EOF'`) for SAIL expressions to avoid quote/brace escaping

## Resource-Specific Notes

### Applications

```bash
# Create and capture UUID
APP=$(echo '{"name":"Case Management","prefix":"CM"}' | appian apps create --format uuids)

# Discover auto-generated objects (groups, folders)
appian apps get $APP | jq '.defaultObjects'
```

### Record Types

```bash
# Create from file
appian rt create --app $APP --file case.json

# Field operations
appian rt fields list <uuid>
echo '{"fieldName":"status","fieldType":"TEXT","length":50}' | appian rt fields add <uuid>

# Relationship operations
appian rt relationships list <uuid>
echo '{"relationshipName":"department","relationshipType":"MANY_TO_ONE",...}' | appian rt relationships add <rt-uuid>

# Record data (CSV)
echo "value
Engineering
Finance" | appian records insert <rt-uuid>
appian records list <rt-uuid>

# Views and actions
appian rt views list <uuid>
appian rt actions list <uuid>
echo '{"displayName":"Create Case","processModelUuid":"<pm-uuid>","actionType":"LIST_ACTION","key":"createCase"}' | appian rt actions add <rt-uuid>

# User filters
appian rt filters list <rt-uuid>
echo '{"name":"Status","facetType":"LIST_OF_VALUES","sourceRef":"<field-uuid>"}' | appian rt filters add <rt-uuid>
```

### Record Events

```bash
# Configure (one-time setup)
echo '{"eventTypes":["Created Order","Approved Order","Shipped Order"]}' \
  | appian rt events configure $RT_UUID --app $APP

# Configure without custom event types (uses defaults)
appian rt events configure $RT_UUID --app $APP

# Get configuration
appian rt events get $RT_UUID

# Find event type IDs
CONFIG=$(appian rt events get $RT_UUID)
LOOKUP_UUID=$(echo $CONFIG | jq -r '.eventTypeLookupRecordTypeUuid')
appian records list $LOOKUP_UUID

# Add event types after initial config
echo "value
Reopened Case" | appian records insert $LOOKUP_UUID
```

### Expression Rules

```bash
# Create
echo '{"name":"CM_GetFullName","inputs":[{"name":"firstName","type":"Text"},{"name":"lastName","type":"Text"}],"expression":"=ri!firstName & \" \" & ri!lastName"}' | appian er create --app $APP

# Test/execute
echo '{"inputs":{"firstName":"Jane","lastName":"Doe"}}' | appian er run <uuid>
```

### Interfaces

```bash
# Create
appian interfaces create --app $APP --file dashboard.json

# Test-render
echo '{"inputs":{"record":null}}' | appian interfaces run <uuid>
```

### Process Models

```bash
# Create (requires PM folder UUID and error alert group)
appian pm create --app $APP --file create-case-pm.json

# Update nodes and variables (full replacement)
appian pm update <uuid> --file updated-pm.json

# Run unattended PM and get results
echo '{"inputs":{"caseRecord":...}}' | appian pm run <uuid>

# Discover node types
appian pm node-types list
appian pm node-types get "internal3.write_records_to_source_23r3"
appian pm node-types get "internal.17" --form $INTERFACE_UUID
appian pm node-types get "internal3.integration" --reference $INTEGRATION_UUID
```

### Sites

```bash
# Create
appian sites create --app $APP --file case-site.json

# Update (pages array is full replacement)
appian sites get $UUID | jq '.pages += [{"name":"Admin","targetUuid":"<uuid>"}]' | appian sites update $UUID
```

### Supporting Objects

```bash
# Constants
echo '{"name":"CM_ADMIN_GROUP","type":"GROUP","value":"CM Administrators"}' | appian constants create --app $APP

# Groups
echo '{"name":"CM Case Managers","parentGroupName":"CM Users"}' | appian groups create --app $APP

# Folders
echo '{"name":"Templates","parentFolderUuid":"<uuid>"}' | appian folders create --app $APP
appian folders contents <uuid>

# Documents
echo '{"name":"template.xlsx","parentFolderUuid":"<uuid>","content":"<content>"}' | appian documents upload --app $APP
appian documents text <uuid>
```

## Common Workflow

A typical application build follows this sequence:

```bash
# 1. Create application
APP=$(echo '{"name":"Case Management"}' | appian apps create --format uuids)

# 2. Discover defaults (auto-generated groups, folders)
appian apps get $APP | jq '.defaultObjects'

# 3. Create record types
appian rt create --app $APP --file case-record-type.json

# 4. Create interfaces
appian interfaces create --app $APP --file case-form.json

# 5. Create process models
appian pm create --app $APP --file create-case-pm.json

# 6. Wire it together (record actions, sites)
appian sites create --app $APP --file case-site.json
```
