# 03 — Response Standard

Version: 3.1
Status: Active
Audience: Backend, Frontend, Mobile, DevOps
Platform: Framework Independent (Laravel, Spring Boot, Node.js, WordPress, Go, Python, etc.)

---

## Overview

Every API response must follow a consistent envelope regardless of endpoint, platform, or framework.

**Goals:**
- Predictable response shape across all services
- Consistent error handling for all clients
- Easy frontend and mobile integration
- Scalable across microservices
- Framework independent
- Backward compatible

---

## Response Envelope

### Success — Single Resource

```json
{
    "success": true,
    "message": "User retrieved successfully.",
    "data": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "name": "Vandet",
        "email": "seanvandet@gmail.com"
    }
}
```

### Success — Collection

```json
{
    "success": true,
    "message": "Users retrieved successfully.",
    "data": [
        { "id": "550e8400-e29b-41d4-a716-446655440000", "name": "Vandet" },
        { "id": "661f9511-f3ac-52e5-b827-557766551111", "name": "John" }
    ]
}
```

### Success — Collection with Pagination

```json
{
    "success": true,
    "message": "Users retrieved successfully.",
    "data": [
        { "id": "550e8400-e29b-41d4-a716-446655440000", "name": "Vandet", "role_id": 2, "status": "active" }
    ],
    "pagination": {
        "current_page": 1,
        "last_page": 5,
        "per_page": 20,
        "total": 82,
        "from": 1,
        "to": 20
    },
    "links": {
        "next": "/api/v1/users?page=2",
        "prev": null,
        "first": "/api/v1/users?page=1",
        "last": "/api/v1/users?page=5"
    }
}
```

### Success — With Reference Data

```json
{
    "success": true,
    "message": "Users retrieved successfully.",
    "data": [
        { "id": "550e8400-e29b-41d4-a716-446655440000", "name": "Vandet", "role_id": 2, "status": "active" }
    ],
    "included": {
        "roles": [
            { "id": 1, "name": "Administrator" },
            { "id": 2, "name": "Editor" }
        ],
        "statuses": [
            { "value": "active",   "label": "Active" },
            { "value": "inactive", "label": "Inactive" }
        ]
    }
}
```

### Success — Create (201)

```json
{
    "success": true,
    "message": "User created successfully.",
    "data": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "name": "Vandet",
        "email": "seanvandet@gmail.com"
    }
}
```

### Error — Validation (422)

```json
{
    "success": false,
    "message": "Validation failed.",
    "code": "VALIDATION_FAILED",
    "errors": {
        "email": ["Email is required."],
        "password": ["Minimum 8 characters."]
    }
}
```

### Error — Business / Domain (409)

```json
{
    "success": false,
    "message": "Email already registered.",
    "code": "USER_EMAIL_DUPLICATE",
    "errors": {}
}
```

### Error — Not Found (404)

```json
{
    "success": false,
    "message": "User not found.",
    "code": "USER_NOT_FOUND",
    "errors": {}
}
```

### Error — Unauthorized (401)

```json
{
    "success": false,
    "message": "Token has expired.",
    "code": "AUTH_TOKEN_EXPIRED",
    "errors": {}
}
```

### Error — Forbidden (403)

```json
{
    "success": false,
    "message": "You do not have permission to perform this action.",
    "code": "AUTH_USER_FORBIDDEN",
    "errors": {}
}
```

### Delete — No Content (204)

```
HTTP/1.1 204 No Content
(empty body)
```

---

## Field Reference

### Success Fields

| Field        | Required | Type             | Description                              |
|--------------|----------|------------------|------------------------------------------|
| `success`    | Yes      | boolean          | Always `true` for success responses      |
| `message`    | Yes      | string           | Human-readable description of the result |
| `data`       | Yes      | object or array  | Main resource or collection              |
| `included`   | No       | object           | Reference/lookup data keyed by type      |
| `pagination` | No       | object           | Present only on paginated collections    |
| `links`      | No       | object           | Navigation URLs, returned with pagination|
| `meta`       | No       | object           | Technical metadata only                  |

### Error Fields

| Field     | Required | Type    | Description                              |
|-----------|----------|---------|------------------------------------------|
| `success` | Yes      | boolean | Always `false` for error responses       |
| `message` | Yes      | string  | Human-readable description of the error  |
| `code`    | Yes      | string  | Machine-readable error code              |
| `errors`  | Yes      | object  | Field-level validation errors or `{}`    |
| `meta`    | No       | object  | Technical metadata (e.g. `request_id`)   |

---

## Field Rules

### `success`
- Always a boolean.
- `true` on success, `false` on any error.
- Never infer success from HTTP status alone — always check this field.

### `message`
- Human-readable. Write as a complete sentence.
- Good: `"User created successfully."` / `"Validation failed."`
- Bad: `"OK"` / `"Done"` / `"Success"`
- Not intended for display to end users — use `errors` for that.

### `data`
- Single resource: return an **object** `{}`.
- Collection: return an **array** `[]`.
- Empty collection: return `[]`, never `null`.
- Never return `null` for `data`.
- **`data` must not be present on error responses** (where `success` is `false`).
  Error responses contain only `success`, `message`, `code`, `errors`, and optionally `meta`.

```json
// Correct error response — no data field
{
    "success": false,
    "message": "User not found.",
    "code": "USER_NOT_FOUND",
    "errors": {}
}

// Wrong — data field present on error
{
    "success": false,
    "message": "User not found.",
    "code": "USER_NOT_FOUND",
    "errors": {},
    "data": null
}
```

#### POST action with no resource result

For POST actions that return no meaningful resource (e.g. logout, verify-email, send notification),
return an empty object:

```json
{
    "success": true,
    "message": "Logged out successfully.",
    "data": {}
}
```

#### 207 Exception

`207 Multi-Status` (bulk partial failure) is the only case where `data` appears alongside
`success: false`. This is because the data describes which items succeeded and which failed,
and that information is essential for clients to handle the partial result.

```json
{
    "success": false,
    "message": "Some items failed.",
    "code": "BULK_PARTIAL_FAILURE",
    "data": {
        "created": 1,
        "failed": 1,
        "items": [
            { "index": 0, "success": true,  "id": "550e8400-e29b-41d4-a716-446655440000" },
            { "index": 1, "success": false, "code": "USER_EMAIL_DUPLICATE", "message": "Email already registered." }
        ]
    },
    "errors": {}
}
```

### `included`
- An object keyed by resource type: `roles`, `statuses`, `countries`, etc.
- Use to bundle reference/lookup data the client needs to render the response.
- Avoids multiple API roundtrips when rendering forms or tables.
- Request via query parameter: `GET /users?include=roles,statuses`
- Omit the field entirely when not requested.

### `pagination`
- Return only on paginated collection endpoints.
- Never include on single-resource endpoints.

| Field          | Type    | Description                  |
|----------------|---------|------------------------------|
| `current_page` | integer | Current page number          |
| `last_page`    | integer | Total number of pages        |
| `per_page`     | integer | Items per page               |
| `total`        | integer | Total number of records      |
| `from`         | integer | First record index this page |
| `to`           | integer | Last record index this page  |

### `links`
- Always returned together with `pagination`.
- `null` for unavailable directions (e.g. `"prev": null` on page 1).
- Use **relative paths** by default: `/api/v1/users?page=2`
- If your platform requires absolute URLs, use the full URL and document this in your OpenAPI spec. Be consistent within a service — never mix relative and absolute URLs in the same response.
- **Preserve all active query parameters** — filter, sort, search, and `per_page` values must be included in every link. If the request was `GET /users?status=active&sort=-created_at&page=1`, the `next` link must be `/api/v1/users?status=active&sort=-created_at&page=2`, not just `/api/v1/users?page=2`.

### `meta`
- Technical metadata only — never business data.
- Good: `request_id`, `trace_id`, `api_version`, `execution_time_ms`
- Bad: `roles`, `statuses`, `departments` → those belong in `included`
- Recommended minimal shape:

```json
"meta": {
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "execution_time_ms": 42
}
```

### `code` (errors only)
- Machine-readable. Used by clients to handle errors programmatically.
- Format: `DOMAIN_ENTITY_REASON`
- All uppercase, underscored.

### `errors` (errors only)
- Field-level errors keyed by field name. Each value is an array of strings.
- Return `{}` when there are no field-level errors (e.g. business errors).
- Never omit this field on error responses.

---

## Error Code Standard

### Format

```
{DOMAIN}_{ENTITY}_{REASON}
```

| Segment  | Description                       | Example                     |
|----------|-----------------------------------|-----------------------------|
| DOMAIN   | Functional area or service        | `AUTH`, `USER`, `ORDER`     |
| ENTITY   | The resource involved             | `TOKEN`, `EMAIL`, `PAYMENT` |
| REASON   | What went wrong (past-tense verb) | `EXPIRED`, `DUPLICATE`, `NOT_FOUND` |

> **Note on two-segment codes:** `VALIDATION_FAILED` and `SERVER_UNAVAILABLE` use two
> segments because they are generic cross-entity codes where an entity segment adds no
> useful information. This is a documented exception to the three-segment rule.

### Standard Code Catalogue

| Code                       | HTTP | Description                         |
|----------------------------|------|-------------------------------------|
| `VALIDATION_FAILED`        | 422  | Request payload failed validation   |
| `AUTH_TOKEN_EXPIRED`       | 401  | JWT or session token has expired    |
| `AUTH_TOKEN_INVALID`       | 401  | Token is malformed or tampered      |
| `AUTH_USER_UNAUTHORIZED`   | 401  | No credentials provided             |
| `AUTH_USER_FORBIDDEN`      | 403  | Authenticated but lacks permission  |
| `USER_NOT_FOUND`           | 404  | User record does not exist          |
| `USER_EMAIL_DUPLICATE`     | 409  | Email already registered            |
| `RESOURCE_NOT_FOUND`       | 404  | Generic — entity does not exist     |
| `SERVER_UNEXPECTED_ERROR`  | 500  | Unhandled server-side exception     |
| `SERVER_RATE_LIMITED`      | 429  | Request rate limit exceeded         |
| `API_VERSION_MISSING`      | 404  | Request made without a version prefix |
| `API_VERSION_RETIRED`      | 410  | The requested API version has been retired |

> Add new codes to the catalogue before using them. Never invent codes outside the format.

---

## HTTP Status Code Reference

| Operation          | Status | Notes                                    |
|--------------------|--------|------------------------------------------|
| GET                | 200    |                                          |
| POST               | 201    | Resource created                         |
| POST (async)       | 202    | Job accepted, not yet complete           |
| PUT / PATCH        | 200    | Resource updated                         |
| DELETE             | 204    | No body returned                         |
| Bulk partial       | 207    | Some items succeeded, some failed        |
| Validation error   | 422    |                                          |
| Unauthorized       | 401    | Missing or invalid credentials           |
| Forbidden          | 403    | Authenticated but not permitted          |
| Not Found          | 404    |                                          |
| Conflict           | 409    | Duplicate or conflicting state           |
| Rate Limited       | 429    |                                          |
| Server Error       | 500    |                                          |
| Unavailable        | 503    | Service temporarily unavailable          |

**Rules:**
- Never return `200 OK` for an error response.
- `DELETE` always returns `204` with no body — the envelope does not apply.
- Always return `Content-Type: application/json` on all responses that include a body.

---

## Null vs Omitted Fields

Optional fields MUST be omitted when absent. Never return `null`, `{}`, or `[]` as a placeholder.

| Field        | When no data        | Never return         |
|--------------|---------------------|----------------------|
| `included`   | Omit                | `"included": null`   |
| `pagination` | Omit                | `"pagination": null` |
| `links`      | Omit                | `"links": null`      |
| `meta`       | Omit                | `"meta": null`       |
| `data`       | `[]` for collection | `"data": null`       |
| `errors`     | `{}` on errors      | Omit on errors       |

> **Rule of thumb:** envelope fields are omitted when absent; resource data fields use `null`.

---

## Optional Include Parameter

Clients request reference data only when needed, keeping default payloads small.

```
GET /users?include=roles,statuses
```

Without `include`, the `included` field is omitted entirely:
```
GET /users
→ { "success": true, "data": [...] }
```

**Invalid include values** (e.g. `?include=unknown`) are silently ignored — do not return an error. Document available includes per endpoint in the OpenAPI spec.

---

## Design Principles

1. Use HTTP status codes correctly.
2. Keep response envelopes consistent across all endpoints.
3. Separate business data (`data`, `included`) from technical metadata (`meta`).
4. Return only requested data — omit optional fields when absent.
5. Use machine-readable error codes for programmatic handling.
6. Include `request_id` in `meta` to support distributed tracing.
7. Never return `200 OK` for errors.
8. Keep APIs backward compatible — add fields, never remove or rename.
9. Always return `Content-Type: application/json` on all responses that include a body.

---

## Platform Implementation Guide

This standard is framework-independent. The pattern is the same on every platform:
centralize response building in one place so controllers stay clean.

### Laravel (PHP)

```php
return ResponseFactory::success($user, 'User retrieved successfully.');
return ResponseFactory::created($user, 'User created successfully.');
return ResponseFactory::paginated($users, 'Users retrieved successfully.');
return ResponseFactory::validationError($errors);
return ResponseFactory::notFound('USER_NOT_FOUND', 'User not found.');
return ResponseFactory::deleted(); // 204 No Content
```

### Spring Boot (Java)

```java
return ResponseFactory.success(user, "User retrieved successfully.");
return ResponseFactory.created(user, "User created successfully.");
return ResponseFactory.paginated(users, page, "Users retrieved successfully.");
return ResponseFactory.validationError(errors);
return ResponseFactory.notFound("USER_NOT_FOUND", "User not found.");
// DELETE: return ResponseEntity.noContent().build();
```

### Node.js / Express

```js
res.success(user, 'User retrieved successfully.');
res.created(user, 'User created successfully.');
res.paginated(users, pagination, 'Users retrieved successfully.');
res.validationError(errors);
res.notFound('USER_NOT_FOUND', 'User not found.');
res.status(204).send(); // DELETE
```

### Go

```go
response.Success(w, user, "User retrieved successfully.")
response.Created(w, user, "User created successfully.")
response.Paginated(w, users, pagination, "Users retrieved successfully.")
response.ValidationError(w, errors)
response.NotFound(w, "USER_NOT_FOUND", "User not found.")
response.NoContent(w) // DELETE
```

### Python / FastAPI or Django

```python
return ResponseFactory.success(user, "User retrieved successfully.")
return ResponseFactory.created(user, "User created successfully.")
return ResponseFactory.paginated(users, pagination, "Users retrieved successfully.")
return ResponseFactory.validation_error(errors)
return ResponseFactory.not_found("USER_NOT_FOUND", "User not found.")
# DELETE: return Response(status_code=204)
```

---

## Changelog

| Version | Date       | Change                                              |
|---------|------------|-----------------------------------------------------|
| 1.0.0   | 2026-06-26 | Initial release                                     |