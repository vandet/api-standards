# API Standards

Version: 1.0
Status: Active
Audience: Backend, Frontend, Mobile, DevOps
Platform: Framework Independent

---

## Overview

This directory contains the complete API standards for all services and platforms.
Every team building or consuming an API must follow these standards.

---

## Why Use This Standard

### Consistency across teams and platforms
Without a shared standard, every developer makes their own decisions — one service returns `data`, another returns `result`, another returns a plain array. Clients end up writing different parsing logic for each service. This standard eliminates that drift.

### Faster frontend and mobile integration
When every response has the same shape, frontend and mobile teams can write one API client layer and reuse it everywhere. No surprises, no custom handling per endpoint.

### Easier debugging and tracing
Standardized `request_id`, `correlation_id`, and `code` fields mean that when something breaks in production, you can trace the exact request across every service without digging through inconsistent logs.

### Onboarding new developers quickly
A new developer joining any service already knows what every response looks like, how errors are structured, and what HTTP status codes to expect. The standard is the documentation.

### Works on any platform
The envelope, error codes, query parameters, and versioning rules are language and framework agnostic. The same standard applies whether you are building in Laravel, Spring Boot, Node.js, Go, Django, or WordPress.

### Backward compatibility by design
The breaking change policy and versioning rules protect existing clients when APIs evolve. Teams can ship new features without breaking anything already in production.

---

## Why Not Use This Standard

Understand the trade-offs before adopting. There are legitimate reasons to skip or adapt parts of this standard.

### You are building a public SDK or library
Public SDKs (npm packages, PyPI libraries) are not REST APIs — they have their own conventions. This standard is for HTTP APIs, not library interfaces.

### You are wrapping a third-party API
If you are building a thin proxy over Stripe, Twilio, or another provider, matching their response shape may be more important than this envelope. Do not force the envelope onto a pass-through proxy.

### Your team is a single developer on a small project
The governance, OpenAPI spec requirements, and review checklist are designed for teams. On a solo project with one consumer, the overhead may not be worth it. Adopt the response envelope and error codes at minimum — skip the governance sections until the team grows.

### You need a streaming or event-driven response
This standard covers request/response HTTP APIs. It does not apply to WebSocket messages, Server-Sent Events (SSE), gRPC streams, or Kafka event schemas. Use it only at the REST API boundary.

### You are migrating a large legacy API
Adopting this standard on an existing API with many consumers requires a version bump and migration guide. It is not a zero-cost change. Plan the migration — do not silently break existing clients.

---

## Document Index

| # | Document | Covers |
|---|----------|--------|
| — | **README.md** | This index |
| 01 | [API Design](01-api-design.md) | REST principles, resource naming, URL conventions, HTTP methods |
| 02 | [Request Standard](02-request-standard.md) | Headers, request body, naming conventions, date format, idempotency |
| 03 | [Response Standard](03-response-standard.md) | Response envelope, field rules, success and error shapes |
| 04 | [Error Code Standard](04-error-code-standard.md) | Error code format, full catalogue, how to register new codes |
| 05 | [Versioning & Deprecation](05-versioning.md) | URL versioning, breaking change policy, deprecation lifecycle |
| 06 | [Authentication](06-authentication.md) | JWT, OAuth2, API keys, RBAC, HTTPS requirements |
| 07 | [Query Standard](07-query-standard.md) | Filtering, sorting, searching, field selection, include, pagination |
| 08 | [API Governance](08-api-governance.md) | Team processes, review checklist, API lifecycle |
| 09 | [OpenAPI Standard](09-openapi-standard.md) | Spec requirements, tooling, documentation rules |
| 10 | [Testing Standard](10-testing-standard.md) | Unit, integration, contract, and E2E testing strategy |
| 11 | [Performance Guidelines](11-performance-guidelines.md) | Latency targets, caching, compression, monitoring |

### Examples

| File | Description |
|------|-------------|
| [examples/authentication.md](examples/authentication.md) | Login, refresh, logout flows |
| [examples/users.md](examples/users.md) | Full CRUD with pagination and reference data |
| [examples/orders.md](examples/orders.md) | Business flow with bulk operations |

---

## Quick Reference

### Response Envelope

```json
// Success
{ "success": true, "message": "...", "data": {} }

// Success — collection
{ "success": true, "message": "...", "data": [], "pagination": {}, "links": {} }

// Error
{ "success": false, "message": "...", "code": "DOMAIN_ENTITY_REASON", "errors": {} }

// DELETE
HTTP 204 No Content — no body
```

### Error Code Format

```
DOMAIN_ENTITY_REASON
AUTH_TOKEN_EXPIRED
USER_EMAIL_DUPLICATE
ORDER_PAYMENT_FAILED
```

### HTTP Status Codes

| Status | When |
|--------|------|
| 200 | GET, PUT, PATCH success |
| 201 | POST success (resource created) |
| 204 | DELETE success |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 409 | Conflict |
| 422 | Validation failed |
| 429 | Rate limited |
| 500 | Server error |

### Request Conventions

```
GET    /api/v1/users
GET    /api/v1/users/{id}
POST   /api/v1/users
PUT    /api/v1/users/{id}
PATCH  /api/v1/users/{id}
DELETE /api/v1/users/{id}

GET /api/v1/users?status=active&sort=-created_at&page=1&per_page=20
GET /api/v1/users?include=roles,statuses
GET /api/v1/users?fields=id,name,email
GET /api/v1/users?search=Vandet
```

---

## Changelog

| Version | Date       | Change |
|---------|------------|--------|
| 1.0     | 2026-06-26 | Initial release |
