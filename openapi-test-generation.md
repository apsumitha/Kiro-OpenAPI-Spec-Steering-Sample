---
inclusion: always
---

# OpenAPI Spec-Driven Test Generation

Guidelines for generating API test cases directly from an OpenAPI (Swagger) specification. Platform-agnostic — adapt to any language, framework, or test runner.

## 1. Parse the Spec First

Before writing any test, extract from each endpoint:
- HTTP method and path
- Path/query/header/body parameters with their types, formats, and constraints (min, max, pattern, enum, required)
- Response status codes and their schemas
- Security requirements (auth schemes)

Use these as the single source of truth. If the spec says `orderId` is an integer between 1 and 10, every test must respect or deliberately violate that constraint.

## 2. Test Categories to Cover Per Endpoint

### Happy Path
- One test per documented success response (200, 201, 204, etc.)
- Use valid parameter values that satisfy all constraints
- Verify response status code, content-type, and body against the response schema

### Parameter Boundary Testing
- For numeric constraints (`minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`):
  - Test at exact boundary values (min, max)
  - Test just inside boundaries (min+1, max-1)
  - Test just outside boundaries (min-1, max+1)
- For string constraints (`minLength`, `maxLength`, `pattern`):
  - Test at exact length limits
  - Test strings that violate the pattern
- For enums: test each valid value and one invalid value
- For arrays (`minItems`, `maxItems`): test at limits and beyond

### Required vs Optional Fields
- Omit each required field individually and expect 400/422
- Omit all optional fields and confirm the request still succeeds
- Send unknown/extra fields and verify the API handles them gracefully

### Invalid Types and Formats
- Send a string where an integer is expected
- Send a malformed date where `date-time` format is required
- Send null for non-nullable fields
- Send empty string for required string fields

### Error Responses
- Test every documented error status code (400, 401, 403, 404, 409, 422, etc.)
- Verify error response bodies match the spec's error schema
- Confirm meaningful error messages are returned

### Authentication and Authorization
- Call protected endpoints without credentials — expect 401
- Call with invalid/expired credentials — expect 401 or 403
- Call with valid credentials but insufficient scope — expect 403

## 3. Inter-Endpoint Dependencies (CRUD Lifecycle)

When endpoints form a lifecycle (create → read → update → delete):
- Test the full lifecycle in sequence
- Verify GET returns what POST created
- Verify PUT/PATCH changes are reflected in subsequent GET
- Verify DELETE makes subsequent GET return 404
- Test DELETE on already-deleted resources (idempotency)

## 4. Response Schema Validation

For every response:
- Validate all required fields are present
- Validate field types match the spec (string, integer, boolean, array, object)
- Validate nested objects and arrays recursively
- Check format constraints (date-time, email, uuid, uri)
- Verify nullable fields can actually be null

## 5. Edge Cases Derived from Spec

- `readOnly` fields: ensure they cannot be set via POST/PUT
- `writeOnly` fields: ensure they do not appear in GET responses
- `deprecated` endpoints: test they still function (or return appropriate warnings)
- Pagination parameters: test first page, last page, out-of-range page
- Rate limiting: if documented, verify 429 responses

## 6. Naming and Organization

- Group tests by API tag or resource (e.g., `Pet API`, `Store API`, `User API`)
- Name tests descriptively: `{METHOD} {path} — {scenario}`
  - Example: `GET /store/order/:id — returns 400 for id outside valid range [1-10]`
- Keep each test independent when possible; document dependencies explicitly

## 7. Prompt Template for Full-Coverage Test Generation

Use the following prompt structure when asking an AI to generate a complete test suite from any OpenAPI spec. Replace the placeholders with your specific values.

```
Generate a full-coverage API test suite from the OpenAPI spec at [SPEC_URL].

## Test Coverage — derive ALL test cases from the spec

For EVERY endpoint defined in the spec, generate tests covering:

### Happy Path
- One test per documented success response using valid data
- Verify status code, response body structure, and field types match the response schema

### Parameter Constraints
Extract minimum, maximum, minLength, maxLength, enum, pattern, and format from the spec and test:
- Boundary values: at min, max, min-1, max+1
- Each valid enum value plus one invalid value
- Invalid formats (string where integer expected, malformed date-time, etc.)

### Required vs Optional Fields
- Omit each required field individually — expect 400/422
- Send only required fields (all optional omitted) — expect success

### Error Responses
- Test every documented error status code per endpoint (400, 404, 405, 422, etc.)
- Verify error response body structure matches the spec

### CRUD Lifecycle
- Full create → read → update → delete → confirm-deletion sequence per resource
- Verify GET reflects POST/PUT changes
- Verify DELETE makes subsequent GET return 404

### Authentication
- Call protected endpoints without credentials — expect 401/403

### Edge Cases
- Deprecated endpoints — test they still respond
- File upload endpoints
- Batch/bulk endpoints
- Special utility endpoints (login, logout, health, etc.)

## Mock Server (if applicable)
- Must enforce the same parameter constraints as the spec
- Must validate required fields and return appropriate error codes
- Must validate enum values
- Must return response bodies matching the spec schemas

## Key Spec Constraints to Enforce
[List the specific constraints from your spec, e.g.:]
- [Resource] [field]: [type], minimum [X], maximum [Y]
- [Resource] [field] enum: [value1], [value2], [value3]
- [Resource] [field]: [type] format [format]
```

## 8. Platform Adaptation Checklist

When applying these guidelines to a specific stack:
- [ ] Choose an HTTP client (axios, fetch, requests, RestAssured, etc.)
- [ ] Choose an assertion library native to the platform
- [ ] Set base URL via config/environment variable
- [ ] Implement auth token setup in a shared fixture or beforeAll hook
- [ ] Use the spec's `example` values as default test data when available
- [ ] Generate or maintain a mock server that enforces the same constraints as the spec
- [ ] Ensure test naming follows `{METHOD} {path} — {scenario}` convention
- [ ] Group tests by API tag or resource
- [ ] Generate a coverage report test-report.md mapping tests back to spec endpoints

