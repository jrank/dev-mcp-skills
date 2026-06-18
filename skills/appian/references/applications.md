# Applications

## Create JSON Schema

```json
{
  "name": "Case Management",
  "prefix": "CM",
  "description": "Optional description"
}
```

- `name` (required) — application display name
- `prefix` (optional) — naming prefix for objects (e.g., "CM"). Auto-generated from name if omitted.
- `description` (optional)

## What Creation Produces

Creating an application auto-generates:
- **PREFIX Administrators** group — developers and admins
- **PREFIX Users** group — all end users (includes Administrators as members)
- Default rule folder, document folder (Knowledge Center), and process model folder

Capture the application UUID from the create response for use in subsequent operations.

## Discovery After Creation

After creating an app, discover its auto-generated objects by getting the application details. Look for the `defaultObjects` field which contains UUIDs for the auto-generated groups, folders, and process model folder.

## Application Prefix Convention

The prefix propagates to all objects in the application:
- Record types: `CM Case`, `CM Customer` (prefix + Title Case singular; plural name has no prefix)
- Interfaces: `CM_Dashboard`, `CM_CaseForm`
- Expression rules: `CM_GetFullName`, `CM_IsEligible`
- Process models: `CM Create Case`, `CM Reassign Case`
- Constants: `CM_ADMIN_GROUP`, `CM_STATUS_OPEN`
- Groups: `CM Administrators`, `CM Case Managers`
- Sites: `CM Case Management` (object name), `Case Management` (display name)
