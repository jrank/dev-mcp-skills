# Appian CLI Tool Reference

## Discovery

The CLI has built-in help at every level:
```
appian --help                        # list all resources
appian <resource> --help             # list actions for a resource
appian <resource> <action> --help    # show flags, JSON schema, examples
```

Use these to discover available commands, flags, and JSON schemas. This reference covers patterns, conventions, and non-obvious behaviors that `--help` doesn't communicate well.

## Setup

The `appian` CLI is bundled with the skill at `./scripts/appian`. Always `cd` into the skill's directory before running commands so relative paths resolve correctly.

Auth is configured in `~/.appian/config.yaml`. The CLI handles credentials automatically — no auth flags needed per-command. Use `appian status` to verify connectivity.

## Command Structure

```
appian <resource> <action> [uuid] [flags]
```

Common resource aliases: `applications` → `apps`, `record-types` → `rt`, `expression-rules` → `er`, `process-models` → `pm`, `documents` → `docs`

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
  "expression": "=a!gridField(label: \"Employees\", data: recordType!...)"
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

Pipe get → modify → update (idiomatic pattern):
```bash
appian rt get $UUID | jq '.description = "Updated"' | appian rt update $UUID
```

Add to an array without losing existing items:
```bash
appian sites get $UUID | jq '.pages += [{"name":"Admin","targetUuid":"<uuid>"}]' | appian sites update $UUID
```

## Error Handling

Non-zero exit codes on failure. JSON error details on stderr.
```bash
appian rt create --app $APP --file body.json || echo "Create failed"
```

## Non-Obvious Behaviors

### Array fields are full replacement
When you update nodes, pages, process variables, or inputs — you replace the entire array. Always get first, modify the array, then update.

### CSV for record data
`appian records insert` and `appian records list` use CSV format (header row + data rows), not JSON. Pipe CSV via stdin or `--file`. Boolean: `1`/`0`. Dates: `YYYY-MM-DD`.

### Process model node type enrichment
`appian pm node-types get` accepts `--form $UUID` (for attended nodes — shows form inputs) and `--reference $UUID` (for integrations/subprocesses — shows their parameters). Without these, you get only the base schema.

### Record events discovery
After configuring events, the event type IDs live in a lookup record type. Chain commands to find them:
```bash
CONFIG=$(appian rt events get $RT_UUID)
LOOKUP_UUID=$(echo $CONFIG | jq -r '.eventTypeLookupRecordTypeUuid')
appian records list $LOOKUP_UUID
```

### Application defaults
After creating an app, discover auto-generated objects (PM folder UUID, group names):
```bash
appian apps get $APP | jq '.defaultObjects'
```

## Key Patterns

- Always list before creating to discover what exists
- Store UUIDs in shell variables for multi-step workflows
- Use `--format uuids` when piping creation output into subsequent commands
- Get an object before updating — especially for array fields
- Use heredoc (`cat << 'EOF'`) for SAIL expressions to avoid quote/brace escaping
- CSV format for record data operations (pipe via stdin or `--file`)

## Common Workflow

A typical application build:

```bash
# 1. Create application and capture UUID
APP=$(echo '{"name":"Case Management","prefix":"CM"}' | appian apps create --format uuids)

# 2. Discover defaults (PM folder, groups)
appian apps get $APP | jq '.defaultObjects'

# 3. Create record types
appian rt create --app $APP --file /tmp/case-rt.json

# 4. Create interfaces
cat << 'EOF' | appian interfaces create --app $APP
{ "name": "CM_CaseForm", "inputs": [...], "expression": "=a!formLayout(...)" }
EOF

# 5. Create process models
appian pm create --app $APP --file /tmp/create-case-pm.json

# 6. Wire together (record actions, sites)
appian sites create --app $APP --file /tmp/case-site.json
```
