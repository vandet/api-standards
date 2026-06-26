# Example — Authentication

Full login, refresh, logout, and password change flows.

---

## Login

**Request**

```
POST /api/v1/auth/login
Content-Type: application/json
Accept: application/json
```

```json
{
    "email": "seanvandet@gmail.com",
    "password": "MySecurePass123!"
}
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "Login successful.",
    "data": {
        "access_token":  "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxIn0.abc",
        "refresh_token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxIn0.xyz",
        "token_type":    "Bearer",
        "expires_in":    900
    }
}
```

**Response — 401 Wrong credentials**

```json
{
    "success": false,
    "message": "The provided credentials are incorrect.",
    "code": "AUTH_USER_UNAUTHORIZED",
    "errors": {}
}
```

**Response — 422 Validation failed**

```json
{
    "success": false,
    "message": "Validation failed.",
    "code": "VALIDATION_FAILED",
    "errors": {
        "email":    ["Email is required."],
        "password": ["Password is required."]
    }
}
```

**Response — 403 Account suspended**

```json
{
    "success": false,
    "message": "Your account has been suspended. Please contact support.",
    "code": "AUTH_USER_SUSPENDED",
    "errors": {}
}
```

---

## Refresh Token

**Request**

```
POST /api/v1/auth/refresh
Authorization: Bearer {refresh_token}
Accept: application/json
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "Token refreshed successfully.",
    "data": {
        "access_token":  "eyJhbGciOiJIUzI1NiJ9.new_access.abc",
        "refresh_token": "eyJhbGciOiJIUzI1NiJ9.new_refresh.xyz",
        "token_type":    "Bearer",
        "expires_in":    900
    }
}
```

**Response — 401 Refresh token expired**

```json
{
    "success": false,
    "message": "Your session has expired. Please log in again.",
    "code": "AUTH_TOKEN_EXPIRED",
    "errors": {}
}
```

---

## Logout

**Request**

```
POST /api/v1/auth/logout
Authorization: Bearer {access_token}
Accept: application/json
```

**Response — 200 OK**

> POST actions that return no resource (logout, verify-email, etc.) return `"data": {}`.
> See [03-response-standard.md](../03-response-standard.md#post-action-with-no-resource-result).

```json
{
    "success": true,
    "message": "Logged out successfully.",
    "data": {}
}
```

---

## Get Authenticated User

**Request**

```
GET /api/v1/auth/me
Authorization: Bearer {access_token}
Accept: application/json
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "User retrieved successfully.",
    "data": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "name": "Vandet",
        "email": "seanvandet@gmail.com",
        "roles": ["admin"],
        "email_verified_at": "2026-01-15T08:00:00Z",
        "created_at": "2026-01-01T00:00:00Z"
    }
}
```

**Response — 401 No token**

```json
{
    "success": false,
    "message": "Authentication required.",
    "code": "AUTH_TOKEN_MISSING",
    "errors": {}
}
```

---

## Change Password

**Request**

```
PATCH /api/v1/auth/password
Authorization: Bearer {access_token}
Content-Type: application/json
Accept: application/json
```

```json
{
    "current_password":      "OldPass123!",
    "password":              "NewSecurePass456!",
    "password_confirmation": "NewSecurePass456!"
}
```

**Response — 200 OK**

```json
{
    "success": true,
    "message": "Password updated successfully.",
    "data": {}
}
```

**Response — 422 Current password wrong**

```json
{
    "success": false,
    "message": "Validation failed.",
    "code": "VALIDATION_FAILED",
    "errors": {
        "current_password": ["The current password is incorrect."]
    }
}
```
