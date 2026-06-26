# 08 — API Governance

Team processes, review checklist, and API lifecycle.

---

## Overview

API governance ensures that all APIs across all services follow the same standards, remain consistent as teams scale, and are managed through a defined lifecycle.

---

## API Design Lifecycle

Every new API endpoint must go through this lifecycle before shipping:

```
Design
  ↓
Review
  ↓
OpenAPI Spec
  ↓
Implementation
  ↓
Testing
  ↓
Deployment
  ↓
Monitoring
  ↓
Deprecation (when applicable)
  ↓
Removal
```

### Design

- Define the resource, URL, HTTP method, request shape, and response shape.
- Check if an existing endpoint already covers the use case.
- Follow [01-api-design.md](01-api-design.md) for naming and structure.

### Review

- At least one peer review before implementation begins.
- Use the API Review Checklist below.
- Breaking changes require a team lead or architect sign-off.

### OpenAPI Spec

- Write the OpenAPI spec before implementing.
- Follow [09-openapi-standard.md](09-openapi-standard.md).

### Implementation

- Follow the architecture layer rules for your platform.
- Use the `ResponseFactory` — never build raw JSON in controllers.

### Testing

- Follow [10-testing-standard.md](10-testing-standard.md).
- Minimum 80% coverage. Feature tests required for every endpoint.

### Deployment

- All CI/CD checks must pass before merge.
- No breaking changes merged without a version bump.

### Monitoring

- Every endpoint is observed from day one.
- Follow [11-performance-guidelines.md](11-performance-guidelines.md).

### Deprecation / Removal

- Follow [05-versioning.md](05-versioning.md).
- Announce before deprecating. Minimum 6-month support period.

---

## API Review Checklist

Complete before every PR merge:

### Design
- [ ] URL follows naming conventions (`/api/v{n}/{resource}`)
- [ ] HTTP method is semantically correct
- [ ] HTTP status codes are correct
- [ ] No breaking changes without a version bump
- [ ] Endpoint documented in OpenAPI spec

### Request
- [ ] Required and optional headers defined
- [ ] Request body uses `snake_case`
- [ ] Input validation is complete
- [ ] Idempotency-Key required for non-idempotent POST

### Response
- [ ] Response uses the standard envelope
- [ ] `success` field present on all responses
- [ ] Error responses include `code` from the catalogue
- [ ] DELETE returns `204 No Content`
- [ ] Optional envelope fields omitted when absent

### Security
- [ ] Authentication enforced
- [ ] Authorization (RBAC) enforced
- [ ] Tenant scoping enforced (if applicable)
- [ ] No secrets or PII in error messages or logs
- [ ] Rate limiting applied

### Testing
- [ ] Unit tests for all Actions / Services
- [ ] Feature tests for all endpoints (happy path + error cases)
- [ ] Coverage ≥ 80%
- [ ] All tests passing in CI

### Documentation
- [ ] OpenAPI spec updated
- [ ] Error codes registered in catalogue
- [ ] Migration guide written (if breaking change)

---

## Error Code Registry Governance

- The error code catalogue lives in [04-error-code-standard.md](04-error-code-standard.md).
- No team may ship a new error code without adding it to the catalogue first.
- Codes are permanent once released — never rename or remove a published code.

---

## Breaking Change Policy

### Allowed (no version bump required)
- Add an optional field to request or response
- Add a new endpoint
- Add an optional query parameter
- Add a new error code to the catalogue
- Improve response message strings

### Not Allowed (requires new version)
- Remove a field from request or response
- Rename a field
- Change a field's data type
- Change an HTTP method for an existing endpoint
- Change URL structure
- Remove an endpoint
- Rename or remove an error code
- Change authentication scheme

### Process for Breaking Changes

1. Create a new API version (e.g. `v2`).
2. Implement the change in the new version.
3. Deprecate the old version following [05-versioning.md](05-versioning.md).
4. Publish a migration guide.

---

## Security Standards

- HTTPS only — no plain HTTP in production.
- JWT signed with RS256 or HS256 (minimum). Rotate signing keys annually.
- OAuth2 for third-party integrations requiring delegated access.
- RBAC enforced at the service layer — not just the gateway.
- Rate limiting on all public endpoints.
- Never expose stack traces, SQL, or internal paths in error responses.
- Secrets in environment variables only — never in code or version control.

See [06-authentication.md](06-authentication.md) for full details.

---

## Logging Standard

Every API request must log:

| Field | Description |
|-------|-------------|
| `request_id` | Unique ID for this request |
| `correlation_id` | Client-provided or server-generated tracing ID |
| `user_id` | Authenticated user ID (or `null` for anonymous) |
| `tenant_id` | Tenant slug (if applicable) |
| `method` | HTTP method |
| `endpoint` | URL path |
| `status` | HTTP status code returned |
| `duration_ms` | Response time in milliseconds |
| `ip` | Client IP address |
| `user_agent` | Client user agent |

**Rules:**
- Never log request bodies containing passwords, tokens, or payment data.
- Never log full Authorization header values.
- Mask PII fields in logs (email → `s***@example.com`).

---

## Monitoring Standard

Every service in production must expose metrics:

| Metric | Description |
|--------|-------------|
| Latency (p50, p95, p99) | Request response time percentiles |
| Error rate | Percentage of 4xx and 5xx responses |
| Throughput | Requests per second |
| Availability | Uptime percentage |

### Distributed Tracing

Use OpenTelemetry for distributed tracing across services.

Recommended tooling:
- **Jaeger** — open source, self-hosted
- **Zipkin** — lightweight alternative
- **Datadog / New Relic** — managed options

Every service must propagate the `X-Correlation-ID` header to downstream calls.

---

## API Ownership

Every API service must have a designated owner:

| Field | Description |
|-------|-------------|
| Owner team | Team responsible for the service |
| On-call contact | Who to contact when the service is down |
| SLA | Uptime and latency commitment |
| Documentation | Link to OpenAPI spec |

This information must be maintained in each service's README.
