---
name: "appian"
description: "Build and modify Appian applications. Covers applications, record types, interfaces, expression rules, process models, sites, Web APIs, constants, groups, folders, and documents. Use this skill whenever working with ANY Appian platform object — including data modeling, relationships, SAIL expressions, forms, dashboards, workflows, deployments, or application structure. Activate for any task that creates, reads, updates, or deletes Appian design objects."
---

## Tool Surface

Appian objects are created and managed through one of two tool surfaces. Check which is available to you:

### Option A: Appian CLI
If the `appian` command is on your PATH, you have the Appian CLI — a noun-verb bash tool. Run `appian --help` to discover resources and actions. Load `references/tools-cli.md` for input/output patterns, conventions, and gotchas that help doesn't cover well.

### Option B: Appian MCP Tools
If your tool list includes Appian design object tools (e.g., `createApplication`, `createRecordType`, `listInterfaces`, `getProcessModel`), you have the Appian MCP server. The server name varies — look for these characteristic tool names regardless of prefix. The tools are self-describing — inspect their parameter schemas directly. Load `references/tools-mcp.md` for usage patterns and conventions.

### Which to use
Use whichever is available. If both are available, prefer MCP tools (structured input/output, no bash escaping concerns). Never mix — pick one surface and use it consistently for the entire task.

---

## Resource Reference Map

Each resource has a dedicated reference file with JSON schemas, design conventions, and pitfalls. Load the relevant reference for your task:

| When to load | Reference File |
|---|---|
| You have the Appian CLI available (`appian` command on PATH) and need input/output patterns | `references/tools-cli.md` |
| You have Appian MCP tools available (e.g., `createApplication`, `createRecordType` in tool list) and need usage patterns | `references/tools-mcp.md` |
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
1. Determine your tool surface: check if Appian MCP tools are available (e.g., `createApplication`, `createRecordType` in your tool list) or if the `appian` CLI is on PATH. Load the matching tool reference for patterns and conventions. For the CLI, also use `appian --help` and `appian <resource> --help` to discover specific commands.
2. Load the primary resource reference (e.g., `references/record-types.md` for record type work)
3. Load supplementary references as needed (e.g., `references/data-modeling.md` for schema design, `references/field-types.md` for type constraints)
4. Load `references/sail.md` when writing SAIL expressions for interfaces
5. Load `references/change-planning.md` when starting a multi-object task to understand dependency ordering

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

## Tips

- Always discover what exists before creating (list before create)
- Get an object before updating it — updates replace provided fields entirely
- Store UUIDs in variables for multi-step workflows
- Record type relationships require both sides declared (MANY_TO_ONE + ONE_TO_MANY)
- All `recordType!` references in expressions must use UUID-qualified format: `'recordType!{uuid}Name.fields.{fieldUuid}fieldName'`
