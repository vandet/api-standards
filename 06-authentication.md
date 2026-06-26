# 06 — Authentication

JWT, service tokens, API keys, RBAC, and HTTPS requirements.

---

## Transport Security

- **HTTPS only** — all API communication must use TLS 1.2 or higher.
- HTTP requests must be rejected or redirected to HTTPS.
- Never transmit tokens, credentials, or sensitive data over plain HTTP.

---

## Authentication Methods

| Method           | Header                                  | Used by                  |
| ---------------- | --------------------------------------- | ------------------------ |
| JWT Bearer token | `Authorization: Bearer {token}`         | End users                |
| Service token    | `Authorization: Bearer {SERVICE_TOKEN}` | Inter-service calls      |
| API key          | `X-API-Key: {key}`                      | Third-party integrations |

---

## JWT (End User Authentication)

### Token Structure

Standard JWT with these claims:

```json
{
    "sub": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": 1,
    "email": "seanvandet@gmail.com",
    "tenant_id": "acme",
    "roles": ["admin", "editor"],
    "iat": 1719393600,
    "exp": 1719480000
}
```

| Claim       | Description                 |
| ----------- | --------------------------- |
| `sub`       | Subject — user UUID (string), follows RFC 7519 |
| `user_id`   | User ID (integer) for internal services        |
| `email`     | User email                  |
| `tenant_id` | Tenant slug for scoping     |
| `roles`     | Array of role names         |
| `iat`       | Issued at (Unix timestamp)  |
| `exp`       | Expiry (Unix timestamp)     |

> **Note on `sub` vs `user_id`:** Both claims are included for compatibility.
> `sub` follows RFC 7519 and is expected by third-party JWT libraries.
> `user_id` is an integer for internal services where integer IDs are required
> (e.g. database foreign keys). New services should prefer `sub`.
> If you migrate to UUID-only IDs in the future, `user_id` can be deprecated
> following the standard versioning policy in [05-versioning.md](./05-versioning.md).

### Token Expiry

| Token type    | Lifetime   |
| ------------- | ---------- |
| Access token  | 15 minutes |
| Refresh token | 30 days    |

### Token Refresh

```
POST /api/v1/auth/refresh
Authorization: Bearer {refresh_token}
```

Response:

```json
{
    "success": true,
    "message": "Token refreshed successfully.",
    "data": {
        "access_token":  "eyJhbGciOiJIUzI1NiJ9...",
        "refresh_token": "eyJhbGciOiJIUzI1NiJ9...",
        "token_type":    "Bearer",
        "expires_in":    900
    }
}
```

### Token Storage (Client Guidance)

| Platform    | Recommended storage                                    |
| ----------- | ------------------------------------------------------ |
| Web (SPA)   | Memory (access token), HttpOnly cookie (refresh token) |
| Mobile      | Secure storage (Keychain / Keystore)                   |
| Server-side | Environment variable or secret manager                 |

Never store tokens in `localStorage` — vulnerable to XSS.

### Common Auth Errors

```json
// 401 — no token
{ "success": false, "message": "Authentication required.", "code": "AUTH_TOKEN_MISSING", "errors": {} }

// 401 — expired token
{ "success": false, "message": "Token has expired.", "code": "AUTH_TOKEN_EXPIRED", "errors": {} }

// 401 — invalid token
{ "success": false, "message": "Token is invalid.", "code": "AUTH_TOKEN_INVALID", "errors": {} }

// 403 — valid token, no permission
{ "success": false, "message": "You do not have permission.", "code": "AUTH_USER_FORBIDDEN", "errors": {} }
```

---

## Service Token (Inter-Service)

Internal microservices authenticate with a shared service token.

```
Authorization: Bearer {SERVICE_TOKEN}
```

- Defined as an environment variable per service.
- Never hardcoded in source code.
- Rotate on a scheduled basis or after suspected exposure.
- Admin routes use service tokens only — no user JWT required.

---

## API Key (Third-Party)

External integrations use long-lived API keys.

```
X-API-Key: sk_live_abc123def456
```

- Prefix convention: `sk_live_` (production), `sk_test_` (sandbox).
- Store as a hashed value in the database — never plaintext.
- Associate each key with a tenant and permission scope.
- Support key rotation without downtime.

---

## Role-Based Access Control (RBAC)

### Model

```
User → has many Roles → each Role has many Permissions
```

### Permission Format

```
{resource}:{action}
```

| Permission       | Description    |
| ---------------- | -------------- |
| `users:read`     | View users     |
| `users:create`   | Create users   |
| `users:update`   | Update users   |
| `users:delete`   | Delete users   |
| `orders:read`    | View orders    |
| `reports:export` | Export reports |

### Enforcement

- Check authentication first — return `401` if no valid token.
- Check authorization second — return `403` if permission is missing.
- Never return `404` to hide the existence of a resource from an unauthorized user (unless resource hiding is a security requirement).

---

## Multi-Tenant Scoping

For multi-tenant systems, every authenticated request must be scoped to a tenant.

```
X-Tenant-ID: acme
```

- Validate that the authenticated user belongs to the tenant in `X-Tenant-ID`.
- All database queries must include a tenant scope — never return cross-tenant data.
- Return `TENANT_NOT_FOUND` if the slug is invalid.
- Return `AUTH_USER_FORBIDDEN` if the user does not belong to the tenant.

---

## Security Checklist

Before any auth-related code is merged:

- [ ] HTTPS enforced
- [ ] Tokens not logged or exposed in error messages
- [ ] JWT signature verified on every request
- [ ] Token expiry enforced
- [ ] Refresh tokens stored securely (hashed)
- [ ] Service tokens in environment variables only
- [ ] API keys hashed in database
- [ ] RBAC permissions checked before data access
- [ ] Tenant scope enforced on all queries
- [ ] Rate limiting on auth endpoints (login, refresh)
- [ ] Brute force protection on login
- [ ] MFA challenge flow implemented if `AUTH_MFA_REQUIRED` code is used
- [ ] MFA challenge endpoint documented in OpenAPI spec (`POST /auth/mfa/verify`)
- [ ] MFA codes are time-limited (TOTP: 30-second window recommended)

---

## Changelog

| Version | Date       | Change                                              |
|---------|------------|-----------------------------------------------------|
| 1.0.0   | 2026-06-26 | Initial release                                     |
