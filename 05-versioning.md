# 05 — Versioning & Deprecation

URL versioning, breaking change policy, and deprecation lifecycle.

---

## Versioning Strategy

APIs use **URL path versioning**.

```
/api/v1/users
/api/v2/users
```

### Why URL Versioning

- Visible in logs, proxies, and API gateways without parsing headers.
- Easy to route at the gateway level.
- Simple for clients to understand and migrate.

### Version Format

- Format: `v{integer}` — `v1`, `v2`, `v3`
- No minor versions in the URL — `v1.1` is not allowed.
- Minor and backward-compatible changes are deployed within the same version.

---

## When to Create a New Version

### Breaking Changes → New Version Required

A breaking change forces clients to update. Always requires a new version.

| Type                         | Example                                     |
| ---------------------------- | ------------------------------------------- |
| Remove a field               | Removing `email` from a user response       |
| Rename a field               | `name` → `full_name`                        |
| Change field type            | `id: integer` → `id: string`                |
| Change HTTP method           | `PUT` → `PATCH` for the same action         |
| Change URL structure         | `/users/{id}/roles` → `/roles?user_id={id}` |
| Change error code            | `USER_DUPLICATE` → `USER_EMAIL_DUPLICATE`   |
| Remove an endpoint           | Removing `GET /users/export`                |
| Change authentication scheme | Sanctum → OAuth2                            |

### Additive Changes → Same Version (No Breaking)

These changes are safe and do not require a new version.

| Type                              | Example                                        |
| --------------------------------- | ---------------------------------------------- |
| Add an optional field             | Adding `avatar_url` to the response            |
| Add a new endpoint                | `GET /users/{id}/activity`                     |
| Add an optional request parameter | Adding `?include=roles`                        |
| Add a new error code              | Adding `USER_PHONE_DUPLICATE` to the catalogue |
| Improve a message string          | Clarifying an error message text               |
| Performance improvements          | Faster response time                           |

---

## Multiple Versions in Parallel

When a new version is released, the old version continues to operate for the full support period.

```
/api/v1/users   ← v1 still active (support period)
/api/v2/users   ← v2 current
```

Teams must maintain both versions until v1 is officially retired.

---

## Deprecation Policy

### Step 1 — Announcement

When a version is deprecated:

1. Update the API documentation to mark the version as deprecated.
2. Add a `Deprecation` response header to all deprecated endpoints:

```
Deprecation: true
Sunset: Sat, 26 Jun 2027 00:00:00 GMT
Link: </api/v2/users>; rel="successor-version"
```

3. Notify all known API consumers via email or in-app notice.

### Step 2 — Support Period

| Version status | Minimum support period     |
| -------------- | -------------------------- |
| Deprecated     | 6 months from announcement |
| End of Life    | After the sunset date      |

### Step 3 — Removal

After the sunset date, return `410 Gone` for all deprecated version endpoints:

```
HTTP/1.1 410 Gone
Location: /api/v2/users
Content-Type: application/json
```

```json
{
    "success": false,
    "message": "This API version has been retired. Please migrate to /api/v2.",
    "code": "API_VERSION_RETIRED",
    "errors": {}
}
```

> **Note:** Some HTTP clients discard the response body on `410` responses.
> Always include the `Location` header pointing to the current version as the
> primary migration signal. The body is supplementary.

After returning `410`:

1. Remove the deprecated routes from the codebase.
2. Update documentation to reflect removal.

---

## Version Lifecycle

```
Active
  ↓  (new version released)
Deprecated  → Announcement + Deprecation headers + 6-month notice
  ↓  (sunset date reached)
Retired  → 410 Gone returned
  ↓  (after retirement period)
Removed  → Routes deleted from codebase
```

---

## Default Version

- Clients must always specify a version explicitly: `/api/v1/users`
- Requests without a version prefix (e.g. `/api/users`) must return `404 Not Found`.
- Never silently redirect unversioned requests to the latest version — this hides version drift from clients and causes subtle breakage during upgrades.

```json
// GET /api/users (no version) → 404
{
    "success": false,
    "message": "API version not specified. Use /api/v1/users.",
    "code": "API_VERSION_MISSING",
    "errors": {}
}
```

> Add `API_VERSION_MISSING` to the error code catalogue under the `API` domain. See [04-error-code-standard.md](./04-error-code-standard.md).

---

## Version in Documentation

Every API reference document must include:

- Current active versions
- Deprecated versions with sunset dates
- Migration guides for breaking changes

---

## Migration Guide Format

When releasing a breaking version, publish a migration guide:

```markdown
## Migrating from v1 to v2

### Changed fields
- `name` → `full_name` (rename)
- `id` type changed from integer to UUID string

### Removed endpoints
- `GET /api/v1/users/export` — use `GET /api/v2/exports?type=users` instead

### New authentication
- v2 requires OAuth2 Bearer token — see [06-authentication.md](./06-authentication.md)
```

---

## Changelog

| Version | Date       | Change                                              |
|---------|------------|-----------------------------------------------------|
| 1.0.0   | 2026-06-26 | Initial release                                     |
