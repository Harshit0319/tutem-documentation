## ADDED Requirements

---

# PRD: API Gateway

**Service:** `api-gateway`
**Port:** 5000 (public-facing; only service exposed outside the private VPC)
**Stack:** Java 17 / Spring Boot 3 (Spring Cloud Gateway)
**Database:** None
**Redis:** None
**Kafka:** None
**Owned collections:** None
**Replicas:** 3 minimum (every request flows through here — no single point of failure)

---

## Module: Request Routing

Routes all inbound HTTP requests to the correct downstream service based on URL path prefix. No business logic lives here — the gateway is a pure proxy.

---

### Feature: Path-Based Proxy Routing

All client traffic enters through the API Gateway. Requests are forwarded to the owning service based on the URL path prefix. Request headers, body, and response status codes are passed through unchanged.

**Proxy route table:**

| Path prefix | Downstream service | Env variable |
|-------------|--------------------|--------------|
| `/api/auth/**` | User Service | `USER_SERVICE_URL` |
| `/api/user/**` | User Service | `USER_SERVICE_URL` |
| `/api/ride-request/**` | Ride Service | `RIDE_SERVICE_URL` |
| `/api/driver/**` | Ride Service | `RIDE_SERVICE_URL` |
| `/api/driver-vehicle/**` | Ride Service | `RIDE_SERVICE_URL` |
| `/api/pairing-ride/**` | Ride Service | `RIDE_SERVICE_URL` |
| `/api/stations/**` | Ride Service | `RIDE_SERVICE_URL` |
| `/api/fare/**` | Ride Service | `RIDE_SERVICE_URL` |
| `/api/fareamount/**` | Ride Service | `RIDE_SERVICE_URL` |
| `/api/vehicle-type/**` | Ride Service | `RIDE_SERVICE_URL` |
| `/api/analytics/**` | Analytics Service | `ANALYTICS_SERVICE_URL` |
| `/iitb-dashboard/**` | Analytics Service | `ANALYTICS_SERVICE_URL` |
| `/bitshyd-dashboard/**` | Analytics Service | `ANALYTICS_SERVICE_URL` |
| `/srinagaradmin/**` | Analytics Service | `ANALYTICS_SERVICE_URL` |
| `/admin/**` | Analytics Service | `ANALYTICS_SERVICE_URL` |
| `/driverTrack/**` | Analytics Service | `ANALYTICS_SERVICE_URL` |
| `/rides/**` | Analytics Service | `ANALYTICS_SERVICE_URL` |
| `/reset-password/**` | Analytics Service | `ANALYTICS_SERVICE_URL` |

**Rules:**
- Routes are matched in the order listed; the first matching prefix wins
- Request headers (including `Authorization`) are forwarded unchanged
- Response status codes and bodies from downstream are returned unchanged
- `/health` is handled locally by the gateway; it is NOT proxied to any downstream service
- The `/internal/**` prefix is blocked at the gateway and never proxied (internal service APIs must not be publicly reachable)

**Configuration environment variables:**

| Variable | Required | Description |
|----------|----------|-------------|
| `USER_SERVICE_URL` | yes | Base URL of User Service e.g. `http://user-service:5001` |
| `RIDE_SERVICE_URL` | yes | Base URL of Ride Service |
| `LOCATION_SERVICE_URL` | yes | Base URL of Location Service |
| `NOTIFICATION_SERVICE_URL` | yes | Base URL of Notification Service |
| `ANALYTICS_SERVICE_URL` | yes | Base URL of Analytics Service |
| `PROXY_TIMEOUT_MS` | yes | Upstream response timeout in ms (default: 5000) |
| `FRONTEND_ORIGIN` | yes | Comma-separated list of allowed CORS origins |

#### Scenario: Request routed to correct service
- **WHEN** a request arrives with path `/api/auth/login`
- **THEN** the gateway forwards the full request to User Service at `USER_SERVICE_URL/api/auth/login`; the response is returned to the client unchanged

#### Scenario: Request headers forwarded intact
- **WHEN** a request with `Authorization: Bearer <token>` is proxied
- **THEN** the downstream service receives the `Authorization` header with the same value; the gateway does not strip or modify it

#### Scenario: Downstream returns non-2xx
- **WHEN** User Service returns HTTP 401
- **THEN** the gateway returns HTTP 401 with the same body to the client; it does not transform error responses

#### Scenario: Internal path blocked
- **WHEN** a request arrives at `/internal/location/nearby-drivers`
- **THEN** the gateway returns HTTP 404; the request is not forwarded to any downstream service

---

### Feature: Unmatched Route Handling

Any request whose path does not match a registered prefix returns a standard error response.

**API Contract:**

| | |
|--|--|
| Response | HTTP 404 `{ "success": false, "message": "route not found", "data": null }` |

#### Scenario: Unmatched path
- **WHEN** a request arrives at a path with no matching prefix (e.g. `/api/unknown`)
- **THEN** HTTP 404 is returned with `{ "success": false, "message": "route not found", "data": null }`; no downstream service is called

---

### Feature: Upstream Unavailable Handling

When a downstream service is unreachable, the gateway returns a structured 502 without crashing.

**API Contract (on upstream unavailable):**

```json
{
  "success": false,
  "message": "upstream service unavailable",
  "data": null
}
```

- HTTP 502 is returned
- All other service routes continue to function normally
- `/health` continues to return HTTP 200 regardless of downstream availability

**On upstream timeout (exceeds `PROXY_TIMEOUT_MS`):**
- HTTP 504 is returned with `{ "success": false, "message": "upstream request timed out", "data": null }`
- The dangling upstream connection is closed; no connection leak

#### Scenario: Downstream service is down
- **WHEN** Ride Service is unreachable and a request arrives for `/api/ride-request/**`
- **THEN** HTTP 502 is returned with `{ "success": false, "message": "upstream service unavailable", "data": null }`; User Service routes continue to work normally

#### Scenario: All downstream services are down
- **WHEN** all downstream services are unreachable simultaneously
- **THEN** each route returns HTTP 502 independently; `GET /health` on the gateway itself still returns HTTP 200

#### Scenario: Downstream service recovers
- **WHEN** a previously unreachable service comes back online
- **THEN** the gateway resumes routing to it on the next request without requiring a restart

#### Scenario: Upstream timeout
- **WHEN** a downstream service does not respond within `PROXY_TIMEOUT_MS`
- **THEN** HTTP 504 is returned; the upstream connection is closed cleanly; the timeout event is logged

---

## Module: CORS

Enforces Cross-Origin Resource Sharing policy on all inbound requests before proxying.

---

### Feature: CORS Enforcement

**CORS configuration:**

| Setting | Value |
|---------|-------|
| Allowed origins | `FRONTEND_ORIGIN` env variable (comma-separated list); each origin MUST include the scheme prefix (`https://`) |
| Allowed methods | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `OPTIONS` |
| Allowed headers | `Content-Type`, `Authorization` |
| Exposed headers | None |
| Credentials | `true` |
| Max age | 3600 seconds (preflight cache) |

**Rules:**
- Preflight `OPTIONS` requests are handled by the gateway and return HTTP 200 with CORS headers; they are NOT forwarded to downstream services
- Requests from origins not in the allowed list return HTTP 403; the request is not proxied
- Origins without an `https://` scheme prefix are invalid and SHALL be rejected at startup (logged as a configuration error)

#### Scenario: Allowed origin — preflight
- **WHEN** a browser sends an `OPTIONS` preflight from an allowed origin
- **THEN** HTTP 200 is returned with correct `Access-Control-Allow-*` headers; no downstream call is made

#### Scenario: Disallowed origin
- **WHEN** a request arrives from an origin not in the allowed list
- **THEN** HTTP 403 is returned; the request is not proxied

#### Scenario: Origin without scheme prefix
- **WHEN** `FRONTEND_ORIGIN` contains `localhost:5000` instead of `https://localhost:5000`
- **THEN** the gateway logs a configuration error at startup; the invalid origin is excluded from the allowed list; requests from that origin receive HTTP 403

---

## Module: Rate Limiting

In-process (per-instance) rate limiting in Phase 1. Distributed rate limiting via Redis is a Phase 2 concern.

---

### Feature: Per-Route Rate Limiting

**Rate limits (in-memory, per gateway instance):**

| Route pattern | Limit | Window |
|---------------|-------|--------|
| `POST /api/auth/mpin-login` | 10 requests | 1 minute per IP |
| `POST /api/auth/send-otp` | 5 requests | 10 minutes per IP |
| `POST /api/auth/forgot-password` | 3 requests | 1 hour per IP |
| All other routes | 200 requests | 1 minute per IP |

On limit exceeded: HTTP 429 `{ "success": false, "message": "rate limit exceeded", "data": null, "meta": { "retryAfterSeconds": <number> } }`.

#### Scenario: OTP rate limit exceeded
- **WHEN** the same IP sends more than 5 OTP requests within 10 minutes
- **THEN** HTTP 429 is returned with `{ "success": false, "message": "rate limit exceeded", "data": null, "meta": { "retryAfterSeconds": <remaining window seconds> } }`; the request is not proxied

#### Scenario: General rate limit
- **WHEN** an IP exceeds 200 requests per minute on any non-auth route
- **THEN** HTTP 429 is returned; subsequent requests within the window continue to return 429

---

## Module: Service Health

---

### Feature: Gateway Health Check

**API Contract:**

| | |
|--|--|
| Method + Path | `GET /health` |
| Auth | Public |
| 200 | `{ "status": "ok", "service": "api-gateway" }` |
| 503 | `{ "status": "starting", "service": "api-gateway" }` |

The health endpoint is handled locally by the gateway. It is not proxied. It returns HTTP 200 as long as the gateway process is running and environment variables are loaded, regardless of downstream service availability.

Used by Kubernetes readiness probe. Returns 503 during startup until initialisation is complete.

#### Scenario: Gateway healthy
- **WHEN** GET /health is called on a running gateway with env loaded
- **THEN** HTTP 200 is returned with `{ "status": "ok", "service": "api-gateway" }`

#### Scenario: Gateway starting up
- **WHEN** GET /health is called before the gateway has finished loading configuration
- **THEN** HTTP 503 is returned; Kubernetes readiness probe marks the pod as not ready

#### Scenario: All downstreams down — gateway still healthy
- **WHEN** all downstream services are unreachable but the gateway process is running
- **THEN** GET /health still returns HTTP 200; downstream health is not a factor in gateway health

---

## Folder Structure Note

API Gateway follows the standard hexagonal layout with a single `routing` module (no business domain modules):

```
src/main/java/com/tutem/apigateway/
├─ modules/
│  └─ routing/
│     ├─ adapter/in/web/         ← Health check controller
│     └─ adapter/out/proxy/      ← Spring Cloud Gateway route definitions
├─ shared/
│  ├─ constants/                 ← Path prefix constants, service name strings
│  ├─ errors/                    ← Gateway error response builders
│  └─ logging/                   ← Request/response logging filter
└─ config/
   ├─ CorsConfig.java            ← CORS configuration bean
   ├─ RateLimitConfig.java       ← In-memory rate limit filter
   └─ RouteConfig.java           ← Spring Cloud Gateway route builder
```
