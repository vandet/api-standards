# Example — Users

Full CRUD with pagination, filtering, sorting, reference data, and bulk operations.

---

## List Users

**Request**

```http
GET /api/v1/users?status=active&sort=-created_at&page=1&per_page=20&include=roles,statuses
Authorization: Bearer {token}
Accept: application/json
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "Users retrieved successfully.",
    "data": [
        {
            "id": 1,
            "name": "Vandet",
            "email": "seanvandet@gmail.com",
            "role_id": 2,
            "status": "active",
            "created_at": "2026-06-01T08:00:00Z"
        },
        {
            "id": 2,
            "name": "John Doe",
            "email": "john@example.com",
            "role_id": 1,
            "status": "active",
            "created_at": "2026-05-20T08:00:00Z"
        }
    ],
    "included": {
        "roles": [
            { "id": 1, "name": "Administrator" },
            { "id": 2, "name": "Editor" }
        ],
        "statuses": [
            { "value": "active",   "label": "Active" },
            { "value": "inactive", "label": "Inactive" },
            { "value": "suspended","label": "Suspended" }
        ]
    },
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

---

## Get a User

**Request**

```http
GET /api/v1/users/1
Authorization: Bearer {token}
Accept: application/json
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "User retrieved successfully.",
    "data": {
        "id": 1,
        "name": "Vandet",
        "email": "seanvandet@gmail.com",
        "role_id": 2,
        "status": "active",
        "email_verified_at": "2026-01-15T08:00:00Z",
        "created_at": "2026-06-01T08:00:00Z",
        "updated_at": "2026-06-10T12:00:00Z"
    }
}
```

**Response — 404 Not found**

```json
{
    "success": false,
    "message": "User not found.",
    "code": "USER_NOT_FOUND",
    "errors": {}
}
```

---

## Create a User

**Request**

```http
POST /api/v1/users
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
Idempotency-Key: 7f6b2a1c-3d4e-5f6a-7b8c-9d0e1f2a3b4c
```

```json
{
    "name":     "Vandet",
    "email":    "seanvandet@gmail.com",
    "password": "SecurePass123!",
    "role_id":  2,
    "status":   "active"
}
```

**Response — 201 Created**

```json
{
    "success": true,
    "message": "User created successfully.",
    "data": {
        "id": 1,
        "name": "Vandet",
        "email": "seanvandet@gmail.com",
        "role_id": 2,
        "status": "active",
        "created_at": "2026-06-26T10:30:00Z"
    }
}
```

**Response — 422 Validation failed**

```json
{
    "success": false,
    "message": "Validation failed.",
    "code": "VALIDATION_FAILED",
    "errors": {
        "email":    ["Email is required.", "Email must be a valid email address."],
        "password": ["Minimum 8 characters.", "Must contain at least one uppercase letter."]
    }
}
```

**Response — 409 Email already exists**

```json
{
    "success": false,
    "message": "Email is already registered.",
    "code": "USER_EMAIL_DUPLICATE",
    "errors": {}
}
```

---

## Update a User (PATCH)

**Request**

```http
PATCH /api/v1/users/1
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
```

```json
{
    "status": "inactive"
}
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "User updated successfully.",
    "data": {
        "id": 1,
        "name": "Vandet",
        "email": "seanvandet@gmail.com",
        "role_id": 2,
        "status": "inactive",
        "updated_at": "2026-06-26T11:00:00Z"
    }
}
```

---

## Replace a User (PUT)

**Request**

```http
PUT /api/v1/users/1
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
```

```json
{
    "name":    "Vandet",
    "email":   "seanvandet@gmail.com",
    "role_id": 1,
    "status":  "active"
}
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "User updated successfully.",
    "data": {
        "id": 1,
        "name": "Vandet",
        "email": "seanvandet@gmail.com",
        "role_id": 1,
        "status": "active",
        "updated_at": "2026-06-26T11:00:00Z"
    }
}
```

---

## Delete a User

**Request**

```http
DELETE /api/v1/users/1
Authorization: Bearer {token}
Accept: application/json
```

**Response — 204 No Content**

```
(empty body)
```

**Response — 404 Not found**

```json
{
    "success": false,
    "message": "User not found.",
    "code": "USER_NOT_FOUND",
    "errors": {}
}
```

---

## Bulk Create

**Request**

```http
POST /api/v1/users/bulk
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
```

```json
{
    "items": [
        { "name": "Vandet",  "email": "seanvandet@gmail.com",  "role_id": 2 },
        { "name": "John",  "email": "john@example.com",  "role_id": 2 },
        { "name": "Alice", "email": "alice@example.com", "role_id": 1 }
    ]
}
```

**Response — 201 All created**

```json
{
    "success": true,
    "message": "3 users created successfully.",
    "data": {
        "created": 3,
        "failed": 0,
        "items": [
            { "index": 0, "success": true, "id": 1 },
            { "index": 1, "success": true, "id": 2 },
            { "index": 2, "success": true, "id": 3 }
        ]
    }
}
```

**Response — 207 Partial failure**

```json
{
    "success": false,
    "message": "Some items failed.",
    "code": "BULK_PARTIAL_FAILURE",
    "data": {
        "created": 2,
        "failed": 1,
        "items": [
            { "index": 0, "success": true,  "id": 1 },
            { "index": 1, "success": false, "code": "USER_EMAIL_DUPLICATE", "message": "Email already registered." },
            { "index": 2, "success": true,  "id": 3 }
        ]
    }
}
```

---

## Bulk Delete

**Request**

```http
DELETE /api/v1/users/bulk
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
```

```json
{
    "ids": [1, 2, 3]
}
```

**Response — 204 No Content**

```
(empty body)
```
