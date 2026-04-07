## ADDED Requirements

### Requirement: HTTP method semantics follow RFC 7231

All services SHALL use HTTP methods strictly as defined in RFC 7231 Section 4.

- `GET` — retrieve a resource or collection; no side effects; safe and idempotent
- `POST` — create a new resource or trigger an action; not idempotent
- `PUT` — replace an existing resource in full; idempotent
- `PATCH` — partially update an existing resource; not required to be idempotent
- `DELETE` — remove a resource; idempotent

#### Scenario: GET request has no body

- **WHEN** a client sends a GET request
- **THEN** the server SHALL ignore any request body and SHALL NOT require one

#### Scenario: PUT replaces entire resource

- **WHEN** a client sends PUT with a partial payload
- **THEN** the server SHALL return HTTP 400 if required fields are missing; it SHALL NOT merge partial fields silently

---

### Requirement: Status code contract

All services SHALL use the following HTTP status codes consistently.


| Code | Meaning               | When to use                                                                         |
| ---- | --------------------- | ----------------------------------------------------------------------------------- |
| 200  | OK                    | Successful GET, PUT, PATCH, DELETE                                                  |
| 201  | Created               | Successful POST that creates a resource; response body includes the new resource ID |
| 204  | No Content            | Successful DELETE with no response body                                             |
| 400  | Bad Request           | Validation failure; malformed JSON; missing required field                          |
| 401  | Unauthorized          | Missing or invalid `Authorization` header / JWT                                     |
| 403  | Forbidden             | Authenticated but not permitted to access this resource                             |
| 404  | Not Found             | Resource does not exist                                                             |
| 409  | Conflict              | Duplicate resource; state transition conflict                                       |
| 413  | Payload Too Large     | Upload exceeds size limit                                                           |
| 422  | Unprocessable Entity  | Semantically invalid input (passes syntax check but fails business rules)           |
| 429  | Too Many Requests     | Rate limit exceeded                                                                 |
| 500  | Internal Server Error | Unhandled server-side error                                                         |
| 502  | Bad Gateway           | API Gateway could not reach upstream service                                        |
| 503  | Service Unavailable   | Service starting up or temporarily unavailable                                      |
| 504  | Gateway Timeout       | Upstream service did not respond within timeout                                     |


#### Scenario: Validation error returns 400 with field details

- **WHEN** a required field is missing or malformed
- **THEN** the server SHALL return HTTP 400 with a JSON body matching the error contract

#### Scenario: Auth failure returns 401, not 403

- **WHEN** no `Authorization` header is present or the JWT is invalid/expired
- **THEN** the server SHALL return HTTP 401; it SHALL NOT return 403

---

### Requirement: Error response contract

All services SHALL return errors using a consistent JSON structure.

```json
{
  "success": false,
  "message": "<human readable explanation>",
  "data": null,
  "errors": [
    { "field": "<field_name>", "message": "<description>" }
  ]
}
```






Rules:

- `success` is always `false` for error responses
- `message` is a human-readable string for debugging; clients SHALL NOT render this to end users
- `data` is always `null` for error responses
- `errors` is an array; it is OPTIONAL and SHOULD be included on validation errors (400); omit for 401/403/404/500
- Each entry in `errors` contains `field` (the failing field name) and `message` (description of the issue)
- No additional top-level keys are permitted in the error object

#### Scenario: Single field validation error

- **WHEN** one required field is missing
- **THEN** `errors` SHALL contain exactly one entry with the field name and message

#### Scenario: Multiple field validation errors

- **WHEN** multiple fields fail validation
- **THEN** `errors` SHALL contain one entry per failing field; all errors are returned at once (no fail-fast)

#### Scenario: 500 error does not leak internals

- **WHEN** an unhandled exception occurs
- **THEN** `error` SHALL be `"internal_server_error"` and `message` SHALL NOT include stack traces, file paths, or database error messages

---

### Requirement: Success response contract

All services SHALL return successful responses using a consistent JSON envelope.

Rules:

- `success` is always `true` for success responses
- `message` is a human-readable string describing the outcome
- `data` contains the serialised resource or collection; set to `null` when there is no resource to return (e.g. acknowledgements)
- `meta` is OPTIONAL; used to carry pagination and any future metadata (request ID, rate limit info); omit when not needed
- Resource IDs are returned as strings, not ObjectIds
- Timestamps are returned as ISO 8601 UTC strings: `"2025-03-10T14:23:00.000Z"`

**Single resource — `GET` → `200 OK`:**

```
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "success": true,
  "message": "User fetched successfully",
  "data": {
    "userId": "64a1f3e8c2a4b20017e4f501",
    "name": "Harshit Shahi",
    "phone": "+919876543210",
    "role": "commuter",
    "photoUrl": null,
    "createdAt": "2025-03-10T14:23:00.000Z"
  }
}
```

**Create resource — `POST` → `201 Created`:**

```
HTTP/1.1 201 Created
Content-Type: application/json
```

```json
{
  "success": true,
  "message": "User registered successfully",
  "data": {
    "userId": "64a1f3e8c2a4b20017e4f501"
  }
}
```

**Acknowledgement (no resource body) — `POST` / `PUT` / `PATCH` → `200 OK`:**

```
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "success": true,
  "message": "Device token registered",
  "data": null
}
```

**Delete with no body — `DELETE` → `204 No Content`:**

```
HTTP/1.1 204 No Content
```

*(empty body)*

**Collection list, unpaginated — `GET` → `200 OK`:**

```
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "success": true,
  "message": "Users fetched successfully",
  "data": [
    { "userId": "64a1f3e8c2a4b20017e4f501", "name": "Harshit Shahi" },
    { "userId": "64a1f3e8c2a4b20017e4f502", "name": "Riya Mehta" }
  ]
}
```

**Collection list, paginated — `GET` → `200 OK`:**

```
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "success": true,
  "message": "Users fetched successfully",
  "data": [
    { "userId": "64a1f3e8c2a4b20017e4f501", "name": "Harshit Shahi" }
  ],
  "meta": {
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 245,
      "hasNext": true
    }
  }
}
```

#### Scenario: Single resource response

- **WHEN** a GET request returns one resource
- **THEN** the body SHALL match `{ "success": true, "message": "...", "data": { <resource fields> } }`

#### Scenario: Create response includes ID

- **WHEN** a POST creates a new resource
- **THEN** the server SHALL return HTTP 201 with `{ "success": true, "message": "...", "data": { "<resourceId>": "<id>" } }`

#### Scenario: Acknowledgement response shape

- **WHEN** a POST/PUT/DELETE operation succeeds but there is no resource to return
- **THEN** HTTP 200 is returned with `{ "success": true, "message": "<description>", "data": null }`

---

### Requirement: Pagination contract for collection endpoints

All endpoints that return variable-length collections SHALL support cursor-based or offset-based pagination.

Pagination query parameters:

- `page` — 1-based page number (default: 1)
- `limit` — items per page (default: 20, max: 100)

Paginated response envelope:

```json
{
  "success": true,
  "message": "<description>",
  "data": [...],
  "meta": {
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 245,
      "hasNext": true
    }
  }
}
```

#### Scenario: Default pagination applied when no params provided

- **WHEN** a client calls a collection endpoint without `page` or `limit` params
- **THEN** the server SHALL default to page 1 with limit 20

#### Scenario: Limit cap enforced

- **WHEN** a client requests `limit=500`
- **THEN** the server SHALL cap at 100 and return `limit: 100` in the pagination object

---

### Requirement: Request and response headers

All services SHALL enforce the following header rules.

- `Content-Type: application/json` is required on all POST/PUT/PATCH request bodies
- `Authorization: Bearer <jwt>` is required on all authenticated endpoints
- `Accept: application/json` is assumed; services do not need to support content negotiation
- `X-Request-ID` — optional; if present, the server SHALL echo it back in the response header

#### Scenario: Missing Content-Type on POST

- **WHEN** a POST request is sent without `Content-Type: application/json`
- **THEN** the server SHALL return HTTP 400 `{ "error": "invalid_content_type" }`

---

### Requirement: API versioning strategy

In Phase 1, all API paths are unversioned (e.g. `/api/auth/register`). Versioning is introduced in Phase 2 via URL path prefix (`/api/v2/...`) only when a breaking change is required.

Rules:

- A breaking change is defined as: removing a field from a response, changing a field type, removing an endpoint, or changing an error code
- Non-breaking additions (new optional fields, new endpoints) do NOT require a version bump
- Both versions SHALL be supported for a minimum of one release cycle during a version transition

#### Scenario: Non-breaking field addition

- **WHEN** a new optional field is added to a response object
- **THEN** no version bump is required; existing clients ignore the new field

#### Scenario: Breaking change requires v2 prefix

- **WHEN** a field is removed from a response
- **THEN** the new endpoint SHALL be published at `/api/v2/...`; the v1 endpoint SHALL remain functional for one release cycle

---

### Requirement: Health check endpoint

Every service SHALL expose `GET /health` with a standard response.

```json
{ "status": "ok", "service": "<service-name>" }
```

- Returns HTTP 200 when the service is ready to accept traffic
- Returns HTTP 503 during startup (used by Kubernetes readiness probe)
- The health endpoint is public (no authentication required)

#### Scenario: Service healthy

- **WHEN** GET /health is called on a running service
- **THEN** it returns HTTP 200 `{ "status": "ok", "service": "<name>" }`

#### Scenario: Service starting up

- **WHEN** GET /health is called before the service has finished initialising
- **THEN** it returns HTTP 503

---

### Requirement: Date and ID serialisation

- All dates in request and response bodies SHALL be ISO 8601 UTC strings
- All MongoDB ObjectIds SHALL be serialised as strings
- No Unix timestamps in response bodies
- Boolean fields SHALL be JSON booleans (`true`/`false`), not strings (`"true"`)

#### Scenario: Date field serialisation

- **WHEN** a resource with a date field is returned
- **THEN** the date field SHALL be formatted as `"2025-03-10T14:23:00.000Z"`

#### Scenario: ID serialisation

- **WHEN** a MongoDB document is returned
- **THEN** the `_id` field SHALL be serialised as a string, not a BSON ObjectId object

