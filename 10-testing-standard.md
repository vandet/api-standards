# 10 — Testing Standard

Unit, integration, feature, contract, and E2E testing strategy for APIs.

---

## Minimum Coverage

| Coverage type     | Minimum required |
|-------------------|-----------------|
| Line coverage     | **80%**         |
| Branch coverage   | **70%**         |
| Function coverage | **80%**         |

> **Do not game coverage metrics.** A test that executes code without asserting its
> output does not count as meaningful coverage. Every test must assert at least one of:
> response status code, response body field, error code, or side effect.

Coverage is enforced in CI — PRs below the threshold are blocked from merging.

---

## Test Types

| Type                  | What it tests                       | Tools                             |
| --------------------- | ----------------------------------- | --------------------------------- |
| Unit                  | Individual functions, actions, DTOs | PHPUnit/Pest, JUnit, Jest, pytest |
| Feature / Integration | Full HTTP request → response        | Pest, Spring MockMvc, Supertest   |
| Contract              | API shape matches the spec          | Schemathesis, Portman, Pact       |
| E2E                   | Full user flow across services      | Playwright, Postman/Newman        |

---

## Test-Driven Development (TDD)

All API code must follow TDD:

```
1. Write the test (RED)             — test fails, no implementation yet
2. Write the implementation (GREEN) — minimal code to pass the test
3. Refactor (IMPROVE)               — clean up without breaking the test
```

**Never write implementation before the test.**

---

## Unit Tests

Test individual Actions, Services, DTOs, and Value Objects in isolation.

### What to test

- Input validation logic
- Business rules inside Actions
- DTO construction and mapping
- Value Object invariants
- Edge cases and boundary values

### Example (Pest PHP)

```php
it('throws when email is invalid', function () {
    expect(fn() => new Email('not-an-email'))
        ->toThrow(InvalidArgumentException::class);
});

it('calculates order total correctly', function () {
    $action = new CalculateOrderTotalAction();
    $result = $action->execute([
        ['price' => 100, 'quantity' => 2],
        ['price' => 50,  'quantity' => 1],
    ]);
    expect($result)->toBe(250);
});
```

---

## Feature Tests (HTTP)

Test the full request → response cycle. Every endpoint needs:

1. **Happy path** — valid request returns correct response shape.
2. **Validation errors** — invalid input returns `422` with correct `errors`.
3. **Not found** — missing resource returns `404` with correct `code`.
4. **Unauthorized** — missing/expired token returns `401`.
5. **Forbidden** — valid token, wrong permission returns `403`.

### Example (Pest PHP)

```php
it('returns paginated users', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->getJson('/api/v1/users?page=1&per_page=20');

    $response->assertStatus(200)
        ->assertJsonStructure([
            'success',
            'message',
            'data' => [['id', 'name', 'email']],
            'pagination' => ['current_page', 'last_page', 'per_page', 'total'],
            'links'      => ['next', 'prev', 'first', 'last'],
        ])
        ->assertJson(['success' => true]);
});

it('returns 422 when email is missing', function () {
    $response = $this->postJson('/api/v1/users', [
        'name' => 'Vandet',
    ]);

    $response->assertStatus(422)
        ->assertJson([
            'success' => false,
            'code'    => 'VALIDATION_FAILED',
        ])
        ->assertJsonPath('errors.email.0', 'Email is required.');
});

it('returns 401 without a token', function () {
    $response = $this->getJson('/api/v1/users');

    $response->assertStatus(401)
        ->assertJson(['code' => 'AUTH_TOKEN_MISSING']);
});

it('does not include data field on error responses', function () {
    $response = $this->getJson('/api/v1/users/nonexistent-uuid');

    $response->assertStatus(404)
        ->assertJsonMissing(['data']);
});
```

---

## Response Shape Assertions

Always assert the full envelope structure, not just the data:

```php
// Assert envelope
$response->assertJsonStructure([
    'success',
    'message',
    'data',
]);

// Assert success flag
$response->assertJson(['success' => true]);

// Assert error code
$response->assertJson(['code' => 'USER_NOT_FOUND']);

// Assert field-level errors
$response->assertJsonPath('errors.email.0', 'Email is required.');

// Assert data field absent on errors
$response->assertJsonMissing(['data']);
```

---

## Contract Tests

Contract tests verify that the API response matches the OpenAPI spec. Run these in CI on every PR.

### Tools

| Tool | Purpose | Recommended |
|------|---------|-------------|
| **Schemathesis** | Property-based contract testing against live server | ✅ Primary |
| **Portman** | Converts OpenAPI spec to Postman collection and runs tests | ✅ Alternative |
| **Pact** | Consumer-driven contract testing between microservices | ✅ For microservices |
| ~~Dredd~~ | ~~Runs spec against live server~~ | ❌ No longer maintained |

```shell
# Schemathesis — run against local server
schemathesis run docs/openapi.yaml --base-url=http://localhost:8000

# With authentication
schemathesis run docs/openapi.yaml \
  --base-url=http://localhost:8000 \
  --header="Authorization: Bearer {token}"
```

---

## E2E Tests

End-to-end tests cover full user flows across multiple services.

### Scope

- Authentication flow: login → access protected resource → refresh → logout
- Core business flows: create order → payment → confirmation
- Error recovery flows: expired token → refresh → retry

### Tools

- **Postman/Newman** — run collection against staging environment
- **Playwright** — browser-level E2E if there is a frontend

### Example (Postman/Newman)

```shell
newman run postman/collections/auth-flow.json \
  --environment postman/environments/staging.json \
  --reporters cli,json
```

---

## Test Data

- Use factories for test data generation — never seed with hardcoded values.
- Each test must create its own data — never depend on data created by another test.
- Clean up after each test (use database transactions that roll back).
- Use realistic data — not `test123`, `aaa`, `foo`.

---

## CI Requirements

All of these must pass before a PR can be merged:

- [ ] Unit tests pass
- [ ] Feature tests pass
- [ ] Line coverage ≥ 80%, branch coverage ≥ 70%
- [ ] Contract tests pass (spec matches implementation)
- [ ] No failing assertions in the test suite

```shell
# Laravel
docker compose exec auth-service php artisan test --coverage

# Node.js
npm test -- --coverage

# Spring Boot
./mvnw test

# Python
pytest --cov=app --cov-report=term-missing
```

---

## Checklist

Before marking an endpoint complete:

- [ ] Unit tests for Actions/Services
- [ ] Feature test — happy path
- [ ] Feature test — validation error
- [ ] Feature test — not found (if applicable)
- [ ] Feature test — unauthorized
- [ ] Feature test — forbidden
- [ ] Feature test — error responses have no `data` field
- [ ] Line coverage ≥ 80%
- [ ] All tests passing in CI

---

## Changelog

| Version | Date       | Change                                                                              |
|---------|------------|-------------------------------------------------------------------------------------|
| 1.1     | 2026-06-26 | Replaced Dredd with Schemathesis, specified coverage types (line/branch/function), added data-on-errors test assertion |
| 1.0     | 2026-06-26 | Initial release                                                                     |
