# 02 — Request Standard

Required and optional headers, request body conventions, naming, date format, and idempotency.

---

## Required Headers

Every API request must include these headers:

| Header          | Value              | Description                                 |
| --------------- | ------------------ | ------------------------------------------- |
| `Authorization` | `Bearer {token}`   | Authentication token (JWT or service token) |
| `Content-Type`  | `application/json` | Required on POST, PUT, PATCH                |
| `Accept`        | `application/json` | Expected response format                    |

```
POST /api/v1/users HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: application/json
Accept: application/json
```

---

## Optional Headers

| Header             | Value           | Description                                            |
| ------------------ | --------------- | ------------------------------------------------------ |
| `X-Correlation-ID` | UUID v4         | Client-generated ID to trace a request across services |
| `Idempotency-Key`  | UUID v4         | Prevents duplicate operations on retried requests      |
| `Accept-Language`  | `en`, `km`      | Preferred response language for user-facing messages   |
| `X-Tenant-ID`      | `{tenant_slug}` | Tenant identifier for multi-tenant services            |

### `X-Correlation-ID`

Clients should generate and attach a UUID v4 for every request. This ID flows through all downstream services and appears in logs, making distributed tracing possible.

```
X-Correlation-ID: 550e8400-e29b-41d4-a716-446655440000
```

If the client does not send one, the server generates it and returns it in the response `meta`.

### `Idempotency-Key`

Required for POST requests that create resources or trigger financial operations. If the same key is sent twice, the server returns the original response instead of processing the request again.

```
Idempotency-Key: 7f6b2a1c-3d4e-5f6a-7b8c-9d0e1f2a3b4c
```

Servers must store idempotency keys for a minimum of 24 hours.

**Enforcement:** If an `Idempotency-Key` is absent on a POST endpoint that requires one, the server must reject the request with `422` and the error code `VALIDATION_FAILED`, including a field error on `idempotency_key`. Servers must never silently process a mutation without one on endpoints where it is required.

---

## Request Body

### Format

All request bodies must be valid JSON.

```json
{
    "name": "Vandet",
    "email": "seanvandet@gmail.com",
    "role_id": 2,
    "status": "active"
}
```

### Field Naming

All fields use **`snake_case`**.

```
// Correct
{ "first_name": "Vandet", "phone_number": "+855123456789" }

// Wrong
{ "firstName": "Vandet", "phoneNumber": "+855123456789" }
```

### Date Format

All dates must be ISO 8601 in UTC.

```json
{
    "scheduled_at": "2026-06-26T10:30:00Z",
    "birth_date": "1990-01-15"
}
```

### Booleans

Use `true` / `false` — never `1` / `0` or strings.

```json
{ "is_active": true }
```

### Nulls

Send `null` explicitly to clear a field. Omitting a field means "do not change it" (for PATCH).

```
// PATCH — clears the phone number
{ "phone_number": null }

// PATCH — does not touch phone number
{ "name": "Vandet" }
```

---

## Validation Rules

Servers must validate all inputs at the boundary. Clients should not rely on server-side validation as a substitute for client-side validation.

Common validation responses: see [03-response-standard.md](./03-response-standard.md) and [04-error-code-standard.md](./04-error-code-standard.md).

---

## Bulk Operations

### Bulk Create

```
POST /api/v1/users/bulk
Content-Type: application/json

{
    "items": [
        { "name": "Vandet", "email": "seanvandet@gmail.com" },
        { "name": "John",   "email": "john@example.com" }
    ]
}
```

Response:

```json
{
    "success": true,
    "message": "2 users created successfully.",
    "data": {
        "created": 2,
        "failed": 0,
        "items": [
            { "index": 0, "success": true, "id": "550e8400-e29b-41d4-a716-446655440000" },
            { "index": 1, "success": true, "id": "661f9511-f3ac-52e5-b827-557766551111" }
        ]
    }
}
```

### Bulk Delete

```
DELETE /api/v1/users/bulk
Content-Type: application/json

{
    "ids": [
        "550e8400-e29b-41d4-a716-446655440000",
        "661f9511-f3ac-52e5-b827-557766551111"
    ]
}
```

Response: `204 No Content`

### Partial Failure in Bulk Create

When some items fail and others succeed, return `207 Multi-Status`:

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
    }
}
```

### Partial Failure in Bulk Delete

When some deletions succeed and others fail, return `207 Multi-Status`:

```json
{
    "success": false,
    "message": "Some items could not be deleted.",
    "code": "BULK_PARTIAL_FAILURE",
    "data": {
        "deleted": 2,
        "failed": 1,
        "items": [
            { "index": 0, "id": "550e8400-e29b-41d4-a716-446655440000", "success": true },
            { "index": 1, "id": "661f9511-f3ac-52e5-b827-557766551111", "success": true },
            { "index": 2, "id": "772a0622-04bd-63f6-c938-668877662222", "success": false, "code": "RESOURCE_LOCKED", "message": "Resource is locked." }
        ]
    }
}
```

If **all** deletions fail, return `422` with `BULK_ALL_FAILED`.
If **all** succeed, return `204 No Content` as normal.

> **Note:** `207` responses include `data` alongside `success: false`. This is the only
> case where `data` appears on an error-type response. See [03-response-standard.md](./03-response-standard.md#207-exception).

---

## Idempotency

| Method | Idempotent | Key Required        |
| ------ | ---------- | ------------------- |
| GET    | Yes        | No                  |
| PUT    | Yes        | No                  |
| PATCH  | Yes        | No                  |
| DELETE | Yes        | No                  |
| POST   | No         | Yes (for mutations) |

POST operations that are not idempotent by nature (create, payment, email send) must accept an `Idempotency-Key` header.

---

## Rate Limiting

All APIs enforce rate limiting. Responses include headers:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1719393600
```

When the limit is exceeded:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

```json
{
    "success": false,
    "message": "Too many requests. Please slow down.",
    "code": "SERVER_RATE_LIMITED",
    "errors": {}
}
```

---

## CORS

APIs must define an explicit CORS policy. Never use wildcard `*` in production.

```
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Authorization, Content-Type, Accept, X-Correlation-ID
```

### Multiple Allowed Origins

Maintain an explicit allowlist. Reflect the request origin only if it matches the list. Never reflect arbitrary origins.

```
# Environment config
ALLOWED_ORIGINS=https://app.example.com,https://admin.example.com

# Server logic — pseudocode
if request.origin in ALLOWED_ORIGINS:
    response.header("Access-Control-Allow-Origin", request.origin)
    response.header("Vary", "Origin")
else:
    return no CORS headers
```

> Always include `Vary: Origin` when reflecting dynamic origins so caches don't serve the wrong CORS header to a different origin.

---

## Changelog

| Version | Date       | Change                                              |
|---------|------------|-----------------------------------------------------|
| 1.0.0   | 2026-06-26 | Initial release                                     |
