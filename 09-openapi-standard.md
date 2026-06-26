# 09 — OpenAPI Standard

OpenAPI specification requirements, documentation rules, and tooling.

---

## Overview

Every API endpoint must be documented in an OpenAPI 3.0 (or higher) specification. The spec is written before implementation and kept in sync with the code.

---

## Spec Location

```
{service}/
└── docs/
    └── openapi.yaml
```

Or for a centralized API gateway:

```
api-specs/
├── auth-service.yaml
├── user-service.yaml
├── order-service.yaml
└── README.md
```

---

## Required Spec Fields

Every `openapi.yaml` must include:

```yaml
openapi: 3.0.3

info:
  title: User Service API
  description: Manages users, roles, and permissions.
  version: 1.0.0
  contact:
    name: Backend Team
    email: backend@example.com

servers:
  - url: https://api.example.com/api/v1
    description: Production
  - url: https://staging-api.example.com/api/v1
    description: Staging
  - url: http://localhost:8000/api/v1
    description: Local

tags:
  - name: Users
    description: User management
  - name: Auth
    description: Authentication
```

---

## operationId Convention

Format: `{verb}{Resource}` in camelCase. Must be unique across the entire spec.

| HTTP Method  | Pattern              | Example           |
|--------------|----------------------|-------------------|
| GET (single) | `get{Resource}`      | `getUser`         |
| GET (list)   | `list{Resources}`    | `listUsers`       |
| POST (create)| `create{Resource}`   | `createUser`      |
| PUT          | `replace{Resource}`  | `replaceUser`     |
| PATCH        | `update{Resource}`   | `updateUser`      |
| DELETE       | `delete{Resource}`   | `deleteUser`      |
| POST (action)| `{verb}{Resource}`   | `cancelOrder`, `verifyUserEmail` |

> Never use hyphens, dots, or slashes in `operationId`. Code generators use this value
> as a method name — it must be a valid identifier in all target languages.

---

## Required Endpoint Fields

Every endpoint must include:

```yaml
/users/{id}:
  get:
    tags:
      - Users
    summary: Get a user
    description: Retrieves a single user by their ID.
    operationId: getUser
    parameters:
      - $ref: '#/components/parameters/UserId'
    responses:
      '200':
        description: User retrieved successfully.
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserResponse'
            example:
              success: true
              message: "User retrieved successfully."
              data:
                id: "550e8400-e29b-41d4-a716-446655440000"
                name: "Vandet"
                email: "seanvandet@gmail.com"
      '401':
        $ref: '#/components/responses/Unauthorized'
      '403':
        $ref: '#/components/responses/Forbidden'
      '404':
        $ref: '#/components/responses/NotFound'
    security:
      - BearerAuth: []
```

| Field         | Required             | Description                                         |
| ------------- | -------------------- | --------------------------------------------------- |
| `tags`        | Yes                  | Group endpoint under a tag                          |
| `summary`     | Yes                  | One-line description (≤ 72 characters)              |
| `description` | Yes                  | Full description of what the endpoint does          |
| `operationId` | Yes                  | Unique camelCase identifier (see convention above)  |
| `parameters`  | Yes (if any)         | All path, query, and header parameters              |
| `requestBody` | Yes (POST/PUT/PATCH) | Request body schema and example                     |
| `responses`   | Yes                  | All possible response codes with schema and example |
| `security`    | Yes                  | Authentication requirement                          |

---

## Reusable Components

Define shared schemas, parameters, and responses in `components` to avoid duplication.

```yaml
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  parameters:
    UserId:
      name: id
      in: path
      required: true
      schema:
        type: string
        format: uuid
      description: The user UUID.

    Page:
      name: page
      in: query
      schema:
        type: integer
        default: 1
      description: Page number.

    PerPage:
      name: per_page
      in: query
      schema:
        type: integer
        default: 20
        maximum: 100
      description: Items per page.

  responses:
    Unauthorized:
      description: Authentication required or token is invalid.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            success: false
            message: "Token has expired."
            code: "AUTH_TOKEN_EXPIRED"
            errors: {}

    Forbidden:
      description: Authenticated but lacks permission.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            success: false
            message: "You do not have permission."
            code: "AUTH_USER_FORBIDDEN"
            errors: {}

    NotFound:
      description: Resource not found.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            success: false
            message: "User not found."
            code: "USER_NOT_FOUND"
            errors: {}

    ValidationError:
      description: Validation failed.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    NoContent:
      description: Resource deleted successfully. No body returned.

  schemas:
    SuccessResponse:
      type: object
      required: [success, message, data]
      properties:
        success:
          type: boolean
          example: true
        message:
          type: string
          example: "User retrieved successfully."
        data:
          oneOf:
            - type: object
              description: Single resource response
            - type: array
              description: Collection response
              items:
                type: object

    ErrorResponse:
      type: object
      required: [success, message, code, errors]
      properties:
        success:
          type: boolean
          example: false
        message:
          type: string
          example: "Validation failed."
        code:
          type: string
          example: "VALIDATION_FAILED"
        errors:
          type: object
          additionalProperties:
            type: array
            items:
              type: string

    Pagination:
      type: object
      properties:
        current_page:
          type: integer
        last_page:
          type: integer
        per_page:
          type: integer
        total:
          type: integer
        from:
          type: integer
        to:
          type: integer

    Links:
      type: object
      properties:
        first:
          type: string
          nullable: true
        last:
          type: string
          nullable: true
        next:
          type: string
          nullable: true
        prev:
          type: string
          nullable: true

    PaginatedSuccessResponse:
      allOf:
        - $ref: '#/components/schemas/SuccessResponse'
        - type: object
          properties:
            pagination:
              $ref: '#/components/schemas/Pagination'
            links:
              $ref: '#/components/schemas/Links'
```

---

## Examples

Every request body and response must include a concrete example — not just a schema.

```yaml
requestBody:
  required: true
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/CreateUserRequest'
      example:
        name: "Vandet"
        email: "seanvandet@gmail.com"
        role_id: 2
        status: "active"
```

---

## Rules

- [ ] Every endpoint has a summary, description, and operationId.
- [ ] operationId follows the `{verb}{Resource}` camelCase convention.
- [ ] Every response (success and error) has a schema and example.
- [ ] All error codes used match the catalogue in [04-error-code-standard.md](./04-error-code-standard.md).
- [ ] Shared schemas are defined in `components`, not duplicated.
- [ ] `SuccessResponse.data` uses `oneOf: [object, array]` — not just `object`.
- [ ] Spec is updated before or alongside implementation — never after.
- [ ] Spec is validated and passes linting in CI.

---

## Recommended Tooling

| Tool                  | Purpose                        |
| --------------------- | ------------------------------ |
| **Swagger UI**        | Interactive API explorer       |
| **Redoc**             | Clean documentation rendering  |
| **Stoplight Studio**  | Visual spec editor             |
| **spectral**          | OpenAPI linting in CI          |
| **openapi-generator** | Generate client SDKs from spec |
| **Postman**           | Manual testing and collections |

---

## Validation in CI

Add OpenAPI linting to the CI pipeline:

```yaml
# .github/workflows/api-lint.yml
- name: Lint OpenAPI spec
  run: npx @stoplight/spectral-cli lint docs/openapi.yaml
```

The pipeline must fail if the spec is invalid or missing required fields.

---

## Changelog

| Version | Date       | Change                                              |
|---------|------------|-----------------------------------------------------|
| 1.0.0   | 2026-06-26 | Initial release                                     |
