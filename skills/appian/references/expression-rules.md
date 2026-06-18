# Expression Rules

## Create JSON Schema

```json
{
  "name": "CM_GetFullName",
  "description": "Returns full name by concatenating first and last name with a space.",
  "inputs": [
    {"name": "firstName", "type": "Text", "description": "Person's first name"},
    {"name": "lastName", "type": "Text", "description": "Person's last name"}
  ],
  "expression": "=ri!firstName & \" \" & ri!lastName"
}
```

- `name` (required)
- `description` (optional but recommended)
- `inputs` (optional) — rule input parameters
- `expression` (optional) — the expression body

## When to Create an Expression Rule

Create a rule when:
- Logic is reused across multiple objects
- Logic is complex enough to benefit from a descriptive name
- You need independent testing or documentation
- Encapsulating a calculation, transformation, or query

Use inline expressions instead when:
- Logic is simple and used in only one place
- A named rule would add indirection without clarity

## Naming Conventions

- **Prefix**: Application prefix + underscore (e.g., `CM_`)
- **Pattern**: `PREFIX_DescriptiveName` in PascalCase
- **Name describes what the rule returns or does**
- **Examples**:
  - `CM_GetFullName` — concatenates names
  - `CM_IsEligibleForEscalation` — returns Boolean
  - `CM_FormatCaseReference` — builds formatted string
  - `CM_CalculateRiskScore` — computes score
  - `CM_GetOpenCaseCount` — returns count

## Rule Inputs

Each input needs:
- `name` — camelCase (becomes `ri!` reference in expression body)
- `type` — Appian type name: `Text`, `Number (Integer)`, `Number (Decimal)`, `Boolean`, `Date`, `Date and Time`, `Time`, `User`, `Group`
- `description` (optional)

For record type inputs, use the `typeReference` string from the record type's details (get the record type to find it).

## Expression Body

- Must start with `=`
- Reference inputs via `ri!` prefix matching input names
- Call other rules: `rule!CM_GetFullName(firstName: "Jane", lastName: "Doe")`
- Reference constants: `cons!CM_STATUS_OPEN`
- Reference record types: `recordType!Case` (must use UUID-qualified format)

## Calling Expression Rules

```
rule!CM_GetFullName(firstName: "Jane", lastName: "Doe")
rule!CM_IsEligibleForEscalation(caseRecord: rv!record)
rule!CM_CalculateRiskScore(score: ri!creditScore)
```

Callable from: interfaces, other expression rules, process model script tasks, XOR conditions.

## Conventions

- One responsibility per rule — split complex rules into composable parts
- Describe what it returns: "Returns true if...", "Returns the formatted..."
- camelCase for input names: `firstName`, `caseRecord`, `includeArchived`
- Boolean inputs use question-style: `isUpdate`, `includeArchived`

## Common Pitfalls

- **Plain-text record type names in expressions** — always use UUID-qualified format
- **Mismatched `ri!` names** — references must exactly match input names (typo = runtime error)
- **Forgetting the `=` prefix** — without it, expression is treated as literal text
- **Updating inputs without updating expression** — renamed inputs break `ri!` references
- **Circular references** — A calls B, B calls A = runtime error
- **Not using application prefix** — rules without prefix collide in shared environments
