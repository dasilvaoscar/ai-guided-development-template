---
name: backend-endpoint-docs
description: Document a backend API endpoint. Use when the user wants to create or update documentation for an HTTP endpoint. Generates a structured markdown file covering description, business rules, request/response schema, HTTP examples, Mermaid flow diagram, side effects, and related code references.
---

# Backend Endpoint Documentation

## Purpose

Generate a complete, self-contained documentation file for a single HTTP endpoint. The output is a markdown file placed in `docs/endpoints/` following a consistent structure.

## Step 1 — Gather information

Before writing anything, collect what is needed. Read the relevant source files to extract accurate information — do not guess or invent values.

Collect from the codebase:

| Information | Where to look |
|---|---|
| HTTP method and path | Router/route file |
| Authentication and authorization | Middleware, route config, or use case |
| Request schema and validations | Controller, DTO, or validation schema |
| Business rules | Use case, domain entity, domain service |
| Response shape | Controller response, output DTO |
| Error cases | Use case error throws, domain errors, controller error handler |
| Side effects | Use case (events, emails, cache invalidation) |

If information is missing or ambiguous, ask the user before proceeding.

## Step 2 — Determine the output file path

Documentation files live at:

```
docs/endpoints/[resource]/[method]-[slug].md
```

Examples:
- `docs/endpoints/users/post-create-user.md`
- `docs/endpoints/orders/get-order-by-id.md`
- `docs/endpoints/products/patch-update-product.md`

Create the directory if it does not exist.

## Step 3 — Fill the template

Use the documentation template: @.claude/skills/backend-endpoint-docs/references/template.md

Apply these rules when filling each section:

### Overview table
- **Idempotent**: `Yes` only if calling the endpoint multiple times with the same input produces the same result with no additional side effects (typical for `GET`, `PUT`).

### Business Rules
- Extract every explicit validation and domain invariant from the use case and domain entity.
- Write each rule as a complete sentence stating what is enforced, not how it is implemented.
- Order rules from most general to most specific.

### Request schema table
- Include every field from the DTO, even optional ones.
- Describe validation constraints precisely (e.g. "min: 1, max: 255" not just "string").

### Response — Errors table
- Include only errors this specific endpoint can realistically return.
- Use domain-specific error codes (e.g. `ORDER_ALREADY_CANCELLED`) alongside generic ones.

### HTTP Examples
- Always include at least one success example and one error example per distinct business rule violation.
- Use realistic but fake data (no real emails, tokens, or IDs).
- Truncate long JWT tokens with `...`.

### Flow diagram (Mermaid)
- The diagram must reflect the actual architecture layers present in the project (Controller → UseCase → Domain → Repository → DB).
- Include all relevant `alt` branches for error paths identified in the business rules.
- Keep participant names short and consistent with the codebase naming.

### Side Effects
- Only list side effects that actually exist in the current implementation.
- If there are none, write "None."

### Related
- Link to actual file paths from the codebase, not hypothetical ones.

## Step 4 — Review before saving

Before writing the file, verify:

- [ ] All business rules from the use case are captured.
- [ ] Every error in the errors table has a corresponding HTTP example.
- [ ] The Mermaid diagram includes all `alt` branches for the listed business rules.
- [ ] No field in the request schema is undocumented.
- [ ] File paths in the Related section exist in the codebase.

## Step 5 — Communicate the output

After creating the file:

1. State the file path created.
2. List the business rules captured — ask the user to confirm they are complete.
3. If any information was inferred (not found explicitly in code), flag it clearly so the user can verify.
