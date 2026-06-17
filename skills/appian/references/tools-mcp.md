# Appian MCP Tool Reference

## Overview

Appian platform operations are available as MCP tools. The tools are discovered automatically via the configured MCP server. Tool names follow a pattern like `mcp__appian__<toolName>` (the exact prefix depends on the server configuration — look for characteristic tool names like `createApplication`, `createRecordType`, `listInterfaces`, `getProcessModel` regardless of prefix).

## Available Tools

Tools are organized by resource type. Each accepts JSON parameters directly (no bash, no piping).

### Applications
- `createApplication` — Create a new application
- `getApplication` — Get application details by UUID
- `listApplications` — List applications with optional filtering
- `deleteApplication` — Delete an application
- `listApplicationObjects` — List all objects in an application
- `addObjectsToApplication` — Associate existing objects with an application

### Record Types
- `createRecordType` — Create a record type with fields
- `getRecordType` — Get record type details (fields, relationships, typeReference)
- `listRecordTypes` — List record types in an application
- `updateRecordType` — Update record type properties
- `deleteRecordType` — Delete a record type
- `addRecordTypeField` — Add a field to a record type
- `listRecordTypeFields` — List fields on a record type
- `updateRecordTypeField` — Update a field
- `deleteRecordTypeField` — Delete a field
- `addRecordTypeRelationship` — Add a relationship between record types
- `listRecordTypeRelationships` — List relationships
- `updateRecordTypeRelationship` — Update a relationship
- `deleteRecordTypeRelationship` — Delete a relationship
- `addRecordTypeView` — Add a view/tab to a record type
- `updateRecordTypeView` — Update a view
- `deleteRecordTypeView` — Delete a view
- `listRecordTypeViews` — List views
- `addRecordTypeAction` — Add a record action (LIST_ACTION or RELATED_ACTION)
- `updateRecordTypeAction` — Update a record action
- `deleteRecordTypeAction` — Delete a record action
- `listRecordTypeActions` — List record actions
- `addRecordTypeUserFilter` — Add a user filter
- `updateRecordTypeUserFilter` — Update a user filter
- `deleteRecordTypeUserFilter` — Delete a user filter
- `listRecordTypeUserFilters` — List user filters
- `configureRecordEvents` — Configure record events (audit history)
- `getRecordEventsConfig` — Get record events configuration
- `insertRecordData` — Insert data rows (CSV format)
- `updateRecordData` — Update data rows by primary key (CSV format)
- `deleteRecordData` — Delete data rows by primary key (CSV format)
- `listRecordData` — List record data

### Interfaces
- `createInterface` — Create an interface with SAIL expression
- `getInterface` — Get interface details
- `listInterfaces` — List interfaces in an application
- `updateInterface` — Update interface expression or inputs
- `deleteInterface` — Delete an interface
- `testInterface` — Evaluate an interface and check for runtime errors

### Expression Rules
- `createExpressionRule` — Create an expression rule
- `getExpressionRule` — Get expression rule details
- `listExpressionRules` — List expression rules
- `updateExpressionRule` — Update expression or inputs
- `deleteExpressionRule` — Delete an expression rule
- `testRule` — Execute a rule with test inputs

### Process Models
- `createProcessModel` — Create a process model
- `getProcessModel` — Get process model details
- `listProcessModels` — List process models
- `updateProcessModel` — Update nodes, variables, start form
- `deleteProcessModel` — Delete a process model
- `createProcessModelNode` — Add a node
- `updateProcessModelNode` — Update a node
- `deleteProcessModelNode` — Delete a node
- `listProcessModelNodes` — List nodes in a process model
- `getProcessModelNode` — Get a single node with full config
- `listProcessModelNodeTypes` — List available node types
- `getProcessModelNodeTypeSchema` — Get input/output schema for a node type
- `testProcessModel` — Run an unattended process model

### Sites
- `createSite` — Create a site with pages
- `getSite` — Get site details
- `listSites` — List sites
- `updateSite` — Update site pages or properties
- `deleteSite` — Delete a site

### Supporting Objects
- `createConstant` — Create a constant
- `getConstant` — Get a constant
- `listConstants` — List constants
- `updateConstant` — Update a constant
- `deleteConstant` — Delete a constant
- `createGroup` — Create a group
- `getGroup` — Get a group
- `listGroups` — List groups
- `updateGroup` — Update a group
- `deleteGroup` — Delete a group
- `addGroupMembers` — Add members to a group
- `removeGroupMember` — Remove a member from a group
- `listGroupMembers` — List group members
- `createFolder` — Create a folder
- `getFolder` — Get a folder
- `listFolders` — List folders
- `listFolderContents` — List folder contents
- `updateFolder` — Update a folder
- `deleteFolder` — Delete a folder
- `uploadDocument` — Upload a document
- `getDocument` — Get document metadata
- `listDocuments` — List documents
- `getDocumentText` — Get extracted text content
- `replaceDocumentContent` — Replace document content
- `deleteDocument` — Delete a document

### Web APIs
- `createWebApi` — Create a Web API
- `getWebApi` — Get Web API details
- `listWebApis` — List Web APIs
- `updateWebApi` — Update a Web API
- `deleteWebApi` — Delete a Web API

### Security
- `getObjectSecurity` — Get security role map for any object
- `updateObjectSecurity` — Set security role map

### Validation
- `validateDesignObject` — Validate all expressions on a design object
- `validateExpression` — Validate a raw SAIL expression

## Key Characteristics

- Parameters are passed directly as JSON arguments — no shell, no piping
- Tool responses are always structured JSON with `uuid`, `name`, etc.
- Errors come back as structured JSON with `error` field
- SAIL expressions are passed as string values in parameters — no special escaping beyond JSON string escaping
- CSV format for record data operations (insertRecordData, updateRecordData, deleteRecordData)
- No need to `cd` anywhere — tools work from any context
- UUIDs are always in the JSON response directly
- Tools are self-describing — inspect their parameter schemas for exact field names and types

## Common Patterns

### Capture UUID from creation
The response from any create tool includes `uuid` directly — just reference it in subsequent calls.

### SAIL expressions
Pass as the `expression` parameter string value. No special escaping needed beyond normal JSON string escaping.

### List before create
Always call the list tool first to discover what already exists before creating new objects.

### Get before update
Updates replace provided fields entirely. Get the current state first to understand what you're changing.

### Record type references
When a parameter needs a record type reference (e.g., process variable type), use the `typeReference` string from `getRecordType`.

### Node type discovery
Before configuring unfamiliar node types, call `getProcessModelNodeTypeSchema` with the node type ID to discover required inputs and outputs. Use the `referenceUuid` parameter to enrich the schema with context from a referenced object (e.g., integration inputs).
