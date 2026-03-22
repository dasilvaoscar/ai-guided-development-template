# [METHOD] [/path/to/endpoint]

> [One-sentence summary of what this endpoint does and its purpose in the system.]

---

## Overview

| Field | Value |
|---|---|
| **Method** | `[GET / POST / PUT / PATCH / DELETE]` |
| **Path** | `/api/v1/[resource]` |
| **Authentication** | `[Bearer token / API Key / None]` |
| **Authorization** | `[Required roles or permissions, e.g. "ADMIN" / "Any authenticated user"]` |
| **Idempotent** | `[Yes / No]` |

---

## Business Rules

> Describe the invariants, validations, and domain rules enforced by this endpoint. Each rule must be independently testable.

1. [Rule 1 — e.g. "Email must be unique across all users"]
2. [Rule 2 — e.g. "Password must be at least 8 characters and contain at least one number"]
3. [Rule 3 — e.g. "A user cannot be created if the plan has reached its seat limit"]
4. [Add as many rules as needed]

---

## Request

### Headers

| Header | Required | Description |
|---|---|---|
| `Authorization` | Yes/No | `Bearer <token>` |
| `Content-Type` | Yes/No | `application/json` |
| `[Custom-Header]` | Yes/No | [Description] |

### Path Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `[id]` | `string (UUID)` | Yes | [Description] |

### Query Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `[page]` | `number` | No | `1` | [Description] |
| `[limit]` | `number` | No | `20` | [Description] |

### Body

```json
{
  "[field]": "[type — description]",
  "[field]": "[type — description]"
}
```

#### Body Schema

| Field | Type | Required | Validation | Description |
|---|---|---|---|---|
| `[name]` | `string` | Yes | min: 1, max: 100 | [Description] |
| `[email]` | `string` | Yes | valid email format | [Description] |
| `[age]` | `number` | No | min: 0, max: 150 | [Description] |

---

## Response

### Success

#### `[200 OK / 201 Created / 204 No Content]`

```json
{
  "[field]": "[example value]",
  "[field]": "[example value]"
}
```

| Field | Type | Description |
|---|---|---|
| `[id]` | `string` | [Description] |
| `[createdAt]` | `string (ISO 8601)` | [Description] |

### Errors

| Status | Code | Trigger |
|---|---|---|
| `400` | `VALIDATION_ERROR` | Request body failed schema validation |
| `401` | `UNAUTHORIZED` | Missing or invalid authentication token |
| `403` | `FORBIDDEN` | Authenticated user lacks the required permission |
| `404` | `[RESOURCE]_NOT_FOUND` | [Resource] with the given ID does not exist |
| `409` | `[RESOURCE]_CONFLICT` | [e.g. Email already registered] |
| `422` | `BUSINESS_RULE_VIOLATION` | [Specific rule violated, e.g. seat limit reached] |
| `500` | `INTERNAL_ERROR` | Unexpected server error |

#### Error Response Body

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable description of the error",
    "details": [
      {
        "field": "email",
        "message": "must be a valid email address"
      }
    ]
  }
}
```

---

## HTTP Examples

### Success

```http
POST /api/v1/[resource] HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: application/json

{
  "[field]": "[value]",
  "[field]": "[value]"
}
```

**Response `201 Created`:**

```json
{
  "[field]": "[value]",
  "[field]": "[value]"
}
```

### Error — Validation failure

```http
POST /api/v1/[resource] HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: application/json

{
  "[field]": ""
}
```

**Response `400 Bad Request`:**

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [{ "field": "[field]", "message": "must not be empty" }]
  }
}
```

### Error — [Business rule violation]

```http
[Reproduce the request that triggers the specific business rule error]
```

**Response `[status]`:**

```json
{
  "error": {
    "code": "[ERROR_CODE]",
    "message": "[Description of the rule that was violated]"
  }
}
```

---

## Flow

> Sequence diagram showing the full request lifecycle through the architecture layers.

```mermaid
sequenceDiagram
    actor Client
    participant Controller
    participant UseCase
    participant Domain
    participant Repository
    participant DB

    Client->>Controller: [METHOD] [/path] + body
    Controller->>Controller: validate request schema

    alt validation fails
        Controller-->>Client: 400 VALIDATION_ERROR
    end

    Controller->>UseCase: execute(dto)
    UseCase->>Repository: findBy[X](value)
    Repository->>DB: SELECT ...
    DB-->>Repository: result

    alt [business rule check — e.g. email already taken]
        UseCase-->>Controller: [DomainError]
        Controller-->>Client: 409 [ERROR_CODE]
    end

    UseCase->>Domain: [Entity].create(data)
    Domain-->>UseCase: entity

    UseCase->>Repository: save(entity)
    Repository->>DB: INSERT / UPDATE ...
    DB-->>Repository: ok

    UseCase-->>Controller: OutputDto
    Controller-->>Client: [2xx] response body
```

---

## Side Effects

> List anything this endpoint causes beyond its primary response.

- [e.g. Sends a welcome email via the notification service]
- [e.g. Emits a `user.created` domain event to the message queue]
- [e.g. Invalidates the `users:list` cache]

---

## Related

| What | Link |
|---|---|
| Use case implementation | `src/application/use-cases/[use-case-name].ts` |
| Domain entity | `src/domain/entities/[entity].ts` |
| Repository interface | `src/domain/repositories/[repository].ts` |
| Related endpoint | `[METHOD] /api/v1/[related-path]` |
