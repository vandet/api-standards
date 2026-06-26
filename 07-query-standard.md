# 07 — Query Standard

Filtering, sorting, searching, field selection, include, and pagination parameters.

---

## Overview

All query parameters use `snake_case` and follow consistent naming conventions across all endpoints.

```
GET /api/v1/users?status=active&sort=-created_at&page=1&per_page=20&search=Vandet&fields=id,name,email&include=roles,statuses
```

---

## Filtering

Use query parameters to filter collections by field value.

```
GET /api/v1/users?status=active
GET /api/v1/users?role_id=2
GET /api/v1/users?country=KH
GET /api/v1/orders?status=pending&payment_method=card
```

### Multiple Values

Use comma-separated values to filter by multiple options (OR logic):

```
GET /api/v1/users?status=active,suspended
GET /api/v1/orders?status=pending,processing
```

### Filter Logic

| Scenario | Logic | Example |
|----------|-------|---------|
| Multiple values, same parameter | OR | `?status=active,suspended` → active OR suspended |
| Multiple different parameters | AND | `?status=active&role_id=2` → active AND role 2 |

> **AND within the same field** (e.g. a record must have multiple tags) is not
> supported via query parameters. Use a `POST /search` endpoint with a request
> body for complex filter expressions.

### Range Filters

For date and numeric ranges, use `_from` and `_to` suffixes:

```
GET /api/v1/orders?created_at_from=2026-01-01&created_at_to=2026-06-30
GET /api/v1/products?price_from=10&price_to=100
```

### Filter Naming Rules

- Use the exact field name from the resource: `status`, `role_id`, `country_id`
- For nested relationships: `role.name=Admin` → avoid where possible; use `role_id` instead
- Never use `filter[field]` bracket syntax

---

## Sorting

Use the `sort` parameter. Prefix with `-` for descending order.

```
GET /api/v1/users?sort=name            ascending by name
GET /api/v1/users?sort=-created_at     descending by created_at
GET /api/v1/users?sort=-created_at,name  descending created_at, then ascending name
```

### Rules

- Default sort must be documented per endpoint (typically `-created_at`).
- Multiple sort fields are comma-separated.
- Only allow sorting on indexed fields to prevent slow queries.

---

## Searching

Use the `search` parameter for full-text or keyword search across relevant fields.

```
GET /api/v1/users?search=Vandet
GET /api/v1/products?search=laptop
```

### Rules

- `search` is a global keyword search — it queries multiple fields defined per endpoint.
- For field-specific search, use the filter parameter directly: `?name=Vandet`
- Minimum search length: 2 characters.
- Case-insensitive.
- Document which fields are searched per endpoint in the OpenAPI spec.

---

## Pagination

All collection endpoints must be paginated. Never return unbounded collections.

```
GET /api/v1/users?page=1&per_page=20
```

### Parameters

| Parameter | Default | Maximum | Description |
|-----------|---------|---------|-------------|
| `page` | 1 | — | Page number (1-indexed) |
| `per_page` | 20 | 100 | Items per page |

### Response

```json
{
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

### Rules

- Maximum `per_page` is 100. Requests exceeding this must return `422`.
- `per_page=0` or negative values must return `422`.
- If no `page` is provided, default to `1`.

---

## Cursor Pagination

Use cursor pagination for real-time feeds, large datasets, or when offset is impractical.

```
GET /api/v1/audit-logs?cursor=eyJpZCI6MTAwfQ==&per_page=20
```

### Response

```json
{
    "pagination": {
        "type": "cursor",
        "per_page": 20,
        "count": 20,
        "next_cursor": "eyJpZCI6MTIwfQ==",
        "prev_cursor": null,
        "has_more": true
    },
    "links": {
        "next": "/api/v1/audit-logs?cursor=eyJpZCI6MTIwfQ==",
        "prev": null
    }
}
```

### Rules

- Cursor values are opaque base64 strings — clients must not decode or construct them.
- `total` is omitted in cursor pagination (requires a full count).
- Use when: real-time data, large datasets (>10k rows), or infinite scroll UIs.

---

## Pagination Conflict Rule

Clients must not send both `page` and `cursor` in the same request.
If both are present, the server must return `422`:

```json
{
    "success": false,
    "message": "Validation failed.",
    "code": "VALIDATION_FAILED",
    "errors": {
        "pagination": ["Cannot use 'page' and 'cursor' together. Choose one pagination strategy."]
    }
}
```

---

## Field Selection

Clients may request a subset of fields to reduce payload size.

```
GET /api/v1/users?fields=id,name,email
```

Response only includes the requested fields:

```json
{
    "success": true,
    "message": "Users retrieved successfully.",
    "data": [
        { "id": 1, "name": "Vandet", "email": "seanvandet@gmail.com" }
    ]
}
```

### Rules

- `id` is always returned even if not requested.
- Unknown field names are ignored (do not return an error).
- Document available fields per endpoint in the OpenAPI spec.

---

## Include (Reference Data)

Clients request reference/lookup data only when needed.

```
GET /api/v1/users?include=roles,statuses
```

### Response with `included`

```json
{
    "data": [...],
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

### Without `include`

```json
{ "data": [...] }
```

### Rules

- Without `?include=`, the `included` field is omitted entirely.
- Unknown include keys are ignored (do not return an error).
- Document available includes per endpoint in the OpenAPI spec.
- Common includes: `roles`, `statuses`, `countries`, `departments`, `categories`, `currencies`

---

## Combining Parameters

All parameters can be combined:

```
GET /api/v1/users
  ?status=active
  &role_id=2
  &search=Vandet
  &sort=-created_at
  &page=1
  &per_page=20
  &fields=id,name,email,status
  &include=roles,statuses
```

---

## Parameter Summary

| Parameter | Example | Description |
|-----------|---------|-------------|
| `{field}` | `?status=active` | Filter by field value |
| `{field}_from` | `?created_at_from=2026-01-01` | Range filter start |
| `{field}_to` | `?created_at_to=2026-12-31` | Range filter end |
| `sort` | `?sort=-created_at,name` | Sort fields (prefix `-` for descending) |
| `search` | `?search=Vandet` | Full-text keyword search |
| `page` | `?page=2` | Page number (offset pagination) |
| `per_page` | `?per_page=20` | Items per page |
| `cursor` | `?cursor=abc123` | Cursor (cursor pagination) |
| `fields` | `?fields=id,name,email` | Field selection |
| `include` | `?include=roles,statuses` | Reference data |

---

## Changelog

| Version | Date       | Change          |
|---------|------------|-----------------|
| 1.0     | 2026-06-26 | Initial release |
