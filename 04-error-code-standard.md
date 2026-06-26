# 04 — Error Code Standard

Error code format, full catalogue, and how to register new codes.

---

## Format

All error codes follow this pattern:

```
DOMAIN_ENTITY_REASON
```

| Segment  | Description | Examples |
|----------|-------------|---------|
| `DOMAIN` | Functional area or service | `AUTH`, `USER`, `ORDER`, `PAYMENT`, `TENANT`, `SERVER` |
| `ENTITY` | The resource or subject | `TOKEN`, `EMAIL`, `PASSWORD`, `ROLE`, `PLAN` |
| `REASON` | What went wrong (past-tense or state) | `EXPIRED`, `DUPLICATE`, `NOT_FOUND`, `FORBIDDEN` |

**Rules:**
- All uppercase, underscore-separated.
- Two or three segments only.
- Never invent codes outside this format.
- All codes must be registered in this catalogue before use.
- For generic errors with no specific entity, use two segments: `VALIDATION_FAILED`, `SERVER_UNAVAILABLE`.

---

## Standard Error Catalogue

### Authentication & Authorization

| Code | HTTP | Description |
|------|------|-------------|
| `AUTH_TOKEN_EXPIRED` | 401 | JWT or session token has expired |
| `AUTH_TOKEN_INVALID` | 401 | Token is malformed, tampered, or unrecognizable |
| `AUTH_TOKEN_MISSING` | 401 | No Authorization header provided |
| `AUTH_USER_UNAUTHORIZED` | 401 | Credentials are wrong (wrong password, etc.) |
| `AUTH_USER_FORBIDDEN` | 403 | Authenticated but lacks required permission |
| `AUTH_USER_SUSPENDED` | 403 | Account is suspended |
| `AUTH_USER_UNVERIFIED` | 403 | Account exists but email is not yet verified |
| `AUTH_SESSION_EXPIRED` | 401 | Session has timed out |
| `AUTH_MFA_REQUIRED` | 403 | Multi-factor authentication is required |

### Validation

| Code | HTTP | Description |
|------|------|-------------|
| `VALIDATION_FAILED` | 422 | One or more request fields failed validation |

> **Note:** `USER_EMAIL_INVALID` and `USER_PASSWORD_WEAK` are field-level validation
> errors. They are returned inside the `errors` object when the top-level code is
> `VALIDATION_FAILED`. They do not replace `VALIDATION_FAILED` as the top-level code.
>
> ```json
> {
>     "success": false,
>     "message": "Validation failed.",
>     "code": "VALIDATION_FAILED",
>     "errors": {
>         "email": ["USER_EMAIL_INVALID: Email format is invalid."],
>         "password": ["USER_PASSWORD_WEAK: Minimum 8 characters required."]
>     }
> }
> ```

### User

| Code | HTTP | Description |
|------|------|-------------|
| `USER_NOT_FOUND` | 404 | User record does not exist |
| `USER_EMAIL_DUPLICATE` | 409 | Email is already registered |
| `USER_EMAIL_INVALID` | 422 | Email format is invalid |
| `USER_PASSWORD_WEAK` | 422 | Password does not meet strength requirements |
| `USER_ROLE_NOT_FOUND` | 404 | Specified role does not exist |

### Resource (Generic)

| Code | HTTP | Description |
|------|------|-------------|
| `RESOURCE_NOT_FOUND` | 404 | Requested resource does not exist |
| `RESOURCE_ALREADY_EXISTS` | 409 | Resource already exists (duplicate) |
| `RESOURCE_CONFLICT` | 409 | Resource is in a conflicting state |
| `RESOURCE_LOCKED` | 423 | Resource is locked and cannot be modified |

> **Note on `423 Locked`:** HTTP `423` has limited support in some frameworks and
> API gateways. If your platform does not support it natively, substitute `409 Conflict`
> with `RESOURCE_LOCKED` in the response body. The error code carries the semantic
> meaning — the HTTP status is secondary.

### Bulk Operations

| Code | HTTP | Description |
|------|------|-------------|
| `BULK_PARTIAL_FAILURE` | 207 | Some items succeeded, some failed |
| `BULK_ALL_FAILED` | 422 | All items in the bulk request failed |
| `BULK_LIMIT_EXCEEDED` | 422 | Bulk request exceeds the allowed item limit |

### File & Upload

| Code | HTTP | Description |
|------|------|-------------|
| `FILE_TOO_LARGE` | 422 | File exceeds the maximum allowed size |
| `FILE_TYPE_INVALID` | 422 | File type is not allowed |
| `FILE_NOT_FOUND` | 404 | Referenced file does not exist |
| `FILE_UPLOAD_FAILED` | 500 | File could not be stored |

### Tenant

| Code | HTTP | Description |
|------|------|-------------|
| `TENANT_NOT_FOUND` | 404 | Tenant slug does not match any tenant |
| `TENANT_SUSPENDED` | 403 | Tenant account is suspended |
| `TENANT_MODULE_DISABLED` | 403 | The requested module is not enabled for this tenant |
| `TENANT_PLAN_EXCEEDED` | 403 | Tenant has exceeded their plan limits |

### Order

| Code | HTTP | Description |
|------|------|-------------|
| `ORDER_NOT_FOUND` | 404 | Order does not exist |
| `ORDER_ALREADY_PAID` | 409 | Order has already been paid |
| `ORDER_CANCELLED` | 409 | Order has been cancelled and cannot be modified |
| `ORDER_ITEM_OUT_OF_STOCK` | 409 | One or more items are out of stock |

### Payment

| Code | HTTP | Description |
|------|------|-------------|
| `PAYMENT_NOT_FOUND` | 404 | Payment record does not exist |
| `PAYMENT_FAILED` | 422 | Payment gateway rejected the transaction |
| `PAYMENT_DUPLICATE` | 409 | Payment has already been processed |
| `PAYMENT_REFUND_FAILED` | 422 | Refund could not be processed |
| `PAYMENT_METHOD_INVALID` | 422 | Payment method is not valid or not supported |

### API Versioning

| Code | HTTP | Description |
|------|------|-------------|
| `API_VERSION_MISSING` | 404 | Request made without a version prefix |
| `API_VERSION_RETIRED` | 410 | The requested API version has been retired |

### Async Jobs

| Code | HTTP | Description |
|------|------|-------------|
| `REPORT_GENERATION_FAILED` | 500 | Async report generation job failed |
| `IMPORT_PROCESSING_FAILED` | 500 | Async import processing job failed |

### Server & Infrastructure

| Code | HTTP | Description |
|------|------|-------------|
| `SERVER_UNEXPECTED_ERROR` | 500 | Unhandled server-side exception |
| `SERVER_UNAVAILABLE` | 503 | Service is temporarily unavailable |
| `SERVER_RATE_LIMITED` | 429 | Request rate limit exceeded |
| `SERVER_TIMEOUT` | 504 | Downstream service timed out |
| `SERVER_MAINTENANCE` | 503 | Service is under scheduled maintenance |

---

## How to Register a New Code

1. Check the catalogue above — confirm the code does not already exist.
2. Follow the `DOMAIN_ENTITY_REASON` format.
3. Add the code to this file under the correct domain section.
4. Add the matching HTTP status code.
5. Write a clear one-line description.
6. Update the shared error code constants file in the codebase.

### Example — adding a new code

**Scenario:** Subscription has expired.

```
Domain:  SUBSCRIPTION
Entity:  PLAN
Reason:  EXPIRED

Code: SUBSCRIPTION_PLAN_EXPIRED
HTTP: 403
Description: The tenant's subscription plan has expired.
```

### Shared Constants (platform example)

Define codes as constants so all services import from one source:

```php
// Laravel — Shared/ErrorCodes.php
final class ErrorCodes
{
    const AUTH_TOKEN_EXPIRED        = 'AUTH_TOKEN_EXPIRED';
    const AUTH_TOKEN_INVALID        = 'AUTH_TOKEN_INVALID';
    const AUTH_USER_FORBIDDEN       = 'AUTH_USER_FORBIDDEN';
    const USER_NOT_FOUND            = 'USER_NOT_FOUND';
    const USER_EMAIL_DUPLICATE      = 'USER_EMAIL_DUPLICATE';
    const VALIDATION_FAILED         = 'VALIDATION_FAILED';
    const RESOURCE_NOT_FOUND        = 'RESOURCE_NOT_FOUND';
    const SERVER_UNEXPECTED_ERROR   = 'SERVER_UNEXPECTED_ERROR';
    const TENANT_MODULE_DISABLED    = 'TENANT_MODULE_DISABLED';
}
```

```java
// Spring Boot — ErrorCodes.java
public final class ErrorCodes {
    public static final String AUTH_TOKEN_EXPIRED      = "AUTH_TOKEN_EXPIRED";
    public static final String USER_NOT_FOUND          = "USER_NOT_FOUND";
    public static final String VALIDATION_FAILED       = "VALIDATION_FAILED";
    public static final String SERVER_UNEXPECTED_ERROR = "SERVER_UNEXPECTED_ERROR";
}
```

```js
// Node.js — errorCodes.js
const ErrorCodes = Object.freeze({
    AUTH_TOKEN_EXPIRED:      'AUTH_TOKEN_EXPIRED',
    USER_NOT_FOUND:          'USER_NOT_FOUND',
    VALIDATION_FAILED:       'VALIDATION_FAILED',
    SERVER_UNEXPECTED_ERROR: 'SERVER_UNEXPECTED_ERROR',
});
module.exports = ErrorCodes;
```

---

## Changelog

| Version | Date       | Change          |
|---------|------------|-----------------|
| 1.0     | 2026-06-26 | Initial release |
