# 01 — API Design

REST principles, resource naming, URL conventions, and HTTP method usage.

---

## REST Principles

APIs must follow REST (Representational State Transfer) principles:

1. **Stateless** — Every request contains all the information needed to process it. No session state stored on the server.
2. **Resource-based** — URLs identify resources (nouns), not actions (verbs).
3. **Uniform interface** — Consistent conventions across all endpoints.
4. **HTTP semantics** — Use HTTP methods and status codes for their intended purpose.

---

## Resource Naming

### Use Nouns, Not Verbs

Resources are things, not actions. The HTTP method defines the action.

| Wrong (verb)          | Correct (noun)    |
|-----------------------|-------------------|
| `GET /getUsers`       | `GET /users`      |
| `POST /createUser`    | `POST /users`     |
| `POST /deleteUser/1`  | `DELETE /users/1` |
| `GET /getUserOrders`  | `GET /users/1/orders` |

### Use Plural Nouns

Always use plural for collection resources.

```
/users        not /user
/orders       not /order
/products     not /product
```

### Use Lowercase with Hyphens

URL paths must be lowercase. Use hyphens (`-`) to separate words — never underscores or camelCase.

```
/audit-logs          correct
/auditLogs           wrong
/audit_logs          wrong
```

### Nesting — Max Two Levels

Nested resources express ownership or relationship. Limit nesting to two levels.

```
// Correct
GET /users/{id}/orders
GET /orders/{id}/items

// Avoid — too deep, hard to maintain
GET /users/{id}/orders/{id}/items/{id}/details
```

When nesting becomes too deep, use query parameters instead:

```
GET /items?order_id=42&user_id=1
```

---

## URL Structure

```
https://api.example.com/api/v1/{resource}/{id}/{sub-resource}
```

| Segment       | Description                          | Example           |
|---------------|--------------------------------------|-------------------|
| `api`         | Fixed prefix                         | `api`             |
| `v1`          | API version                          | `v1`, `v2`        |
| `{resource}`  | Plural noun, lowercase, hyphenated   | `users`, `audit-logs` |
| `{id}`        | Resource identifier                  | `1`, `uuid`       |
| `{sub-resource}` | Nested resource                   | `orders`, `roles` |

---

## HTTP Methods

| Method | Purpose | Idempotent | Body |
|--------|---------|------------|------|
| `GET` | Retrieve resource(s) | Yes | No |
| `POST` | Create a new resource | No | Yes |
| `PUT` | Replace a resource entirely | Yes | Yes |
| `PATCH` | Update part of a resource | Yes | Yes |
| `DELETE` | Remove a resource | Yes | No |

### PUT vs PATCH

- `PUT` replaces the entire resource — client must send all fields.
- `PATCH` updates only the provided fields — client sends only what changes.

```json
// PUT /users/1 — must include all fields
{ "name": "Vandet", "email": "seanvandet@gmail.com", "status": "active" }

// PATCH /users/1 — only changed fields
{ "status": "inactive" }
```

### POST for Actions

When an operation does not map to a standard CRUD method, use `POST` with a verb-based path under the resource:

```
POST /orders/{id}/cancel
POST /orders/{id}/refund
POST /users/{id}/verify-email
POST /auth/refresh
POST /auth/logout
```

---

## Standard Endpoints per Resource

```
GET    /api/v1/users           List all users (paginated)
GET    /api/v1/users/{id}      Get a single user
POST   /api/v1/users           Create a user
PUT    /api/v1/users/{id}      Replace a user
PATCH  /api/v1/users/{id}      Update a user
DELETE /api/v1/users/{id}      Delete a user
```

---

## Identifiers

- Prefer **UUID v4** (`550e8400-e29b-41d4-a716-446655440000`) over auto-increment integers for public-facing IDs.
- Never expose internal database IDs directly in URLs if they are sequential integers — they allow enumeration attacks.
- If using integer IDs internally, consider hashing or obfuscating in public URLs.

---

## Naming Convention — Fields

All request and response field names must use **`snake_case`**.

```json
// Correct
{ "first_name": "Vandet", "created_at": "2026-06-26T10:30:00Z" }

// Wrong
{ "firstName": "Vandet", "createdAt": "2026-06-26T10:30:00Z" }
```

**Rationale:** Consistent with PHP/Laravel, Python, and database column naming. Frontend clients can transform at the API gateway or client layer if needed.

---

## Date and Time

All dates and timestamps must use **ISO 8601** format in UTC.

```
2026-06-26T10:30:00Z        datetime (UTC)
2026-06-26                  date only
10:30:00                    time only
```

- Always return timestamps in UTC — never in local time.
- Store and return `created_at` and `updated_at` on all resources.
- Clients are responsible for converting UTC to local time for display.

---

## Boolean Fields

Use `true` / `false` — never `0` / `1` or `"yes"` / `"no"`.

```json
{ "is_active": true, "email_verified": false }
```

Prefix boolean fields with `is_`, `has_`, or `can_` for clarity.

---

## Null Fields

Return `null` explicitly when a field exists but has no value. Do not omit the field.

```json
// Correct — field exists, no value
{ "deleted_at": null }

// Wrong — field omitted
{ }
```

Exception: optional envelope fields (`included`, `pagination`, `links`, `meta`) are omitted when absent. See [03-response-standard.md](03-response-standard.md).
