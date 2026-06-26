# 03 — Response Standard

Response envelope, field rules, all success and error shapes.

---

## Envelope

Every response must follow this envelope. Never return raw models, plain arrays, or unstructured JSON.

### Success — Single Resource

```json
{
    "success": true,
    "message": "User retrieved successfully.",
    "data": {
        "id": 1,
        "name": "Vandet",
        "email": "seanvandet@gmail.com",
        "created_at": "2026-06-26T10:30:00Z"
    }
}
```

### Success — Collection

```json
{
    "success": true,
    "message": "Users retrieved successfully.",
    "data": [
        { "id": 1, "name": "Vandet" },
        { "id": 2, "name": "John" }
    ]
}
```

### Success — Collection with Pagination

```json
{
    "success": true,
    "message": "Users retrieved successfully.",
    "data": [
        { "id": 1, "name": "Vandet", "role_id": 2, "status": "active" }
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
        "first": "/api/v1/users?page=1",
        "last":  "/api/v1/users?page=5",
        "next":  "/api/v1/users?page=2",
        "prev":  null
    }
}
```

### Success — With Reference Data

```json
{
    "success": true,
    "message": "Users retrieved successfully.",
    "data": [
        { "id": 1, "name": "Vandet", "role_id": 2, "status": "active" }
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
        "id": 1,
        "name": "Vandet",
        "email": "seanvandet@gmail.com",
        "created_at": "2026-06-26T10:30:00Z"
    }
}
```

### Success — With Technical Metadata

```json
{
    "success": true,
    "message": "Users retrieved successfully.",
    "data": [...],
    "meta": {
        "request_id": "req_0184A",
        "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
        "trace_id": "trace_8D921",
        "execution_time_ms": 27,
        "api_version": "v1"
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
        "email":    ["Email is required.", "Email must be a valid email address."],
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
    "message": "Token has expired. Please log in again.",
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

### Error — Server Error (500)

```json
{
    "success": false,
    "message": "An unexpected error occurred. Please try again.",
    "code": "SERVER_UNEXPECTED_ERROR",
    "errors": {},
    "meta": {
        "request_id": "req_0184A"
    }
}
```

### Delete — No Content (204)

```
HTTP/1.1 204 No Content
(empty body — envelope does not apply)
```

---

## Field Reference

### Success Fields

| Field        | Required | Type            | Description |
|--------------|----------|-----------------|-------------|
| `success`    | Yes      | boolean         | Always `true` |
| `message`    | Yes      | string          | Human-readable, for developers and logs |
| `data`       | Yes      | object or array | Main resource or collection |
| `included`   | No       | object          | Reference/lookup data keyed by type |
| `pagination` | No       | object          | Present only on paginated collections |
| `links`      | No       | object          | Navigation URLs, returned with pagination |
| `meta`       | No       | object          | Technical metadata only |

### Error Fields

| Field     | Required | Type    | Description |
|-----------|----------|---------|-------------|
| `success` | Yes      | boolean | Always `false` |
| `message` | Yes      | string  | Human-readable error description |
| `code`    | Yes      | string  | Machine-readable error code |
| `errors`  | Yes      | object  | Field-level errors or `{}` |
| `meta`    | No       | object  | Technical metadata (include `request_id`) |

---

## Field Rules

### `success`
- Always a boolean. Never a string or integer.
- `true` on all success responses, `false` on all error responses.

### `message`
- Purpose: developer logs and debugging — not for direct display to end users.
- Write as a complete sentence ending with a period.
- Good: `"User created successfully."` / `"Validation failed."`
- Bad: `"OK"` / `"Done"` / `"Success"` / `"Error"`
- For end-user display, use the `errors` fields or handle by `code` in the client.

### `data`
- Single resource: return an **object** `{}`.
- Collection: return an **array** `[]`.
- Empty collection: return `[]` — never `null`.
- Never return `null` for `data`.

### `data` in error responses

The `data` field must **not** be included in error responses. Error responses
contain only `success`, `message`, `code`, `errors`, and optionally `meta`.

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

### `included`
- Object keyed by resource type: `roles`, `statuses`, `countries`, etc.
- Use to bundle reference/lookup data the client needs to render the response.
- Avoids multiple API roundtrips when rendering forms or tables.
- Request via: `GET /users?include=roles,statuses`
- Omit the field entirely when not requested or not applicable.

### `pagination`

| Field          | Type    | Description |
|----------------|---------|-------------|
| `current_page` | integer | Current page number |
| `last_page`    | integer | Total number of pages |
| `per_page`     | integer | Items per page |
| `total`        | integer | Total number of records |
| `from`         | integer | First record index on this page |
| `to`           | integer | Last record index on this page |

- Return only on paginated collection endpoints.
- Never include on single-resource endpoints.

### `links`
- Always returned together with `pagination`.
- Use `null` for unavailable directions (e.g. `"prev": null` on page 1).

### `links` — URL format

Links use **relative paths** by default:

```json
"links": {
    "next": "/api/v1/users?page=2",
    "prev": null,
    "first": "/api/v1/users?page=1",
    "last": "/api/v1/users?page=5"
}
```

If your platform requires absolute URLs (e.g. API gateway consumers, mobile clients),
return the full URL and document this in your service's OpenAPI spec:

```json
"links": {
    "next": "https://api.example.com/api/v1/users?page=2"
}
```

> Be consistent within a service — never mix relative and absolute URLs in the same response.

### `meta`
- Technical metadata only — never business data.
- Good: `request_id`, `correlation_id`, `trace_id`, `api_version`, `execution_time_ms`
- Bad: `roles`, `statuses`, `departments` → those belong in `included`

### `code` (errors only)
- Machine-readable. Used by clients to handle errors programmatically.
- Format: `DOMAIN_ENTITY_REASON` — see [04-error-code-standard.md](04-error-code-standard.md).
- All uppercase, underscored.

> **Note on `VALIDATION_FAILED`:** This code uses two segments rather than three
> because validation is not tied to a specific entity — it applies to any request
> payload. It is a documented exception to the `DOMAIN_ENTITY_REASON` format.
> Other two-segment codes follow the same logic: `SERVER_UNAVAILABLE`,
> `SERVER_TIMEOUT`. These are generic cross-entity codes where an entity segment
> would add no useful information.

### `errors` (errors only)
- Keyed by field name. Each value is an array of strings.
- Return `{}` when there are no field-level errors (business errors, auth errors).
- Never omit on error responses.

---

## Null vs Omitted Fields

Optional envelope fields must be omitted when not present. Never return `null`, `{}`, or `[]` as a placeholder for an absent optional field.

| Field        | When absent          | Never return         |
|--------------|----------------------|----------------------|
| `included`   | Omit entirely        | `"included": null`   |
| `pagination` | Omit entirely        | `"pagination": null` |
| `links`      | Omit entirely        | `"links": null`      |
| `meta`       | Omit entirely        | `"meta": null`       |
| `data`       | `[]` for collections | `"data": null`       |
| `errors`     | `{}` on errors       | Omit on errors       |

---

## HTTP Status Codes

| Operation        | Status | Notes |
|------------------|--------|-------|
| GET              | 200    | |
| POST             | 201    | Resource created |
| PUT / PATCH      | 200    | Resource updated |
| DELETE           | 204    | No body returned |
| Validation error | 422    | |
| Unauthorized     | 401    | Missing or invalid credentials |
| Forbidden        | 403    | Authenticated but not permitted |
| Not Found        | 404    | |
| Conflict         | 409    | Duplicate or state conflict |
| Bulk partial     | 207    | Multi-status for bulk operations |
| Rate Limited     | 429    | |
| Server Error     | 500    | |

**Rules:**
- Never return `200 OK` for an error response.
- `DELETE` always returns `204` with no body.
- Always return `Content-Type: application/json` on all responses that include a body.
  `DELETE 204` responses have no body and therefore no `Content-Type` header.

---

## Platform Implementation

Controllers must never build raw JSON. Use a centralized factory.

### Laravel

```php
return ResponseFactory::success($user, 'User retrieved successfully.');
return ResponseFactory::created($user, 'User created successfully.');
return ResponseFactory::paginated($users, 'Users retrieved successfully.');
return ResponseFactory::validationError($errors);
return ResponseFactory::notFound('USER_NOT_FOUND', 'User not found.');
return ResponseFactory::deleted();
```

### Spring Boot

```java
return ResponseFactory.success(user, "User retrieved successfully.");
return ResponseFactory.created(user, "User created successfully.");
return ResponseFactory.paginated(users, page, "Users retrieved successfully.");
return ResponseFactory.validationError(errors);
return ResponseFactory.notFound("USER_NOT_FOUND", "User not found.");
return ResponseEntity.noContent().build();
```

### Node.js / Express

```js
res.success(user, 'User retrieved successfully.');
res.created(user, 'User created successfully.');
res.paginated(users, pagination, 'Users retrieved successfully.');
res.validationError(errors);
res.notFound('USER_NOT_FOUND', 'User not found.');
res.status(204).send();
```

### Python / FastAPI

```python
return ResponseFactory.success(user, "User retrieved successfully.")
return ResponseFactory.created(user, "User created successfully.")
return ResponseFactory.paginated(users, pagination, "Users retrieved successfully.")
return ResponseFactory.validation_error(errors)
return ResponseFactory.not_found("USER_NOT_FOUND", "User not found.")
return Response(status_code=204)
```

---

## Changelog

| Version | Date       | Change          |
|---------|------------|-----------------|
| 1.0     | 2026-06-26 | Initial release |
