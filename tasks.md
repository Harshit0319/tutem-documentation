## 1. API Protocol Standard

- [x] 1.1 Create `openspec/changes/platform-standards-and-prd/specs/api-protocol/spec.md` as the canonical HTTP API standard document (RFC 7231 semantics, status code table, error contract, success response examples with HTTP status lines, pagination contract, header rules, versioning strategy, health check standard, date and ID serialisation)
- [x] 1.2 Distribute the API protocol document to all service owners and confirm agreement before any service begins endpoint implementation

## 2. Folder Structure Standard

- [x] 2.1 Create `openspec/changes/platform-standards-and-prd/specs/folder-structure-standard/spec.md` as the canonical folder layout document (hexagonal structure, Java/Spring Boot only, naming conventions table, dependency direction rules, shared code placement, environment profile files)
- [x] 2.2 Verify the User Service scaffold at `services/user-service/` matches the standard; document any deviations

  **Deviation found:** `services/user-service/` is a Node.js scaffold with MVC-style layout (`src/controllers/`, `src/services/`, `src/repositories/`, `src/models/`). This does NOT match the standard.
  The reference implementation at `tutem_services/user-service/` uses the correct Java/Spring Boot hexagonal layout.
  **Required action:** `services/user-service/` must be rebuilt from scratch as a Java 17 / Spring Boot 3 project following the hexagonal structure before any feature work begins (covered by Roadmap Phase 0).

## 3. PRD — API Gateway

- [x] 3.1 Create `specs/prd-api-gateway/spec.md` with Module: Request Routing — proxy route table (all path prefixes + env variables), header passthrough rules, `/internal/**` block, unmatched-route 404, upstream-unavailable 502, timeout 504
- [x] 3.2 Add Module: CORS — allowed origins, `https://` scheme enforcement, preflight local handling, disallowed-origin 403
- [x] 3.3 Add Module: Rate Limiting — per-route in-memory limits table (login, OTP, forgot-password, default), 429 response with `retryAfterSeconds`
- [x] 3.4 Add Module: Service Health — `/health` local handling, 200 regardless of downstream state, 503 during startup
- [x] 3.5 Add folder structure note for API Gateway's single-module hexagonal shape

## 4. PRD — User Service

- [x] 4.1 Create `specs/prd-user-service/spec.md` with Module: Authentication — User Registration (API + `user_registered` Kafka event), OTP Send/Verify, MPIN Login (set + login), JWT Refresh
- [x] 4.2 Add Module: User Profile — Get/Update Profile (role-immutability rule), Device Token Registration (idempotent + `device_token_registered` Kafka event), Shuttle Opt-in/Opt-out
- [x] 4.3 Add Module: Admin — Admin Login (JWT with `role: admin` claim)
- [x] 4.4 Add `users` MongoDB schema table (all fields, types, required flags, indexes)

## 5. PRD — Ride Service

- [x] 5.1 Create `specs/prd-ride-service/spec.md` with Module: Ride Lifecycle — Book Ride (API + `ride_created` Kafka event), Driver Assignment cron (nearby-drivers cross-service call + `ride_assigned` / `no_driver_available` events), Accept/Decline (API + `ride_accepted` event), Status Transitions (driver-reached, start-trip, complete with `ride_completed` event), Cancel Ride (`ride_cancelled` event), Stale Cleanup cron
- [x] 5.2 Add Module: Driver Management — Vehicle register/update API, Driver Search Log (written by cron + `analytics.events` event)
- [x] 5.3 Add Module: Fare Management — Fare configuration admin CRUD, fare calculation on completion (formula + fallback)
- [x] 5.4 Add Module: Station Queue (FIFO) — commuter check-in, driver arrival + FIFO match logic, leave queue
- [x] 5.5 Add Module: User-to-User Pairing — create/cancel pairing request, OSMnx subprocess match, failure handling
- [x] 5.6 Add MongoDB schemas: `riderequests`, `drivervehicledetails`, `fares` (with index tables)

## 6. PRD — Location Service

- [x] 6.1 Create `specs/prd-location-service/spec.md` with Module: Driver Location Tracking — `clientEvent` Socket.io handler (validate → MongoDB upsert → Redis GEOADD → Kafka publish), connect/disconnect lifecycle, stale cleanup cron
- [x] 6.2 Add Module: Live Ride Tracking — `joinTrackingRoom` / `leaveTrackingRoom` events, 200ms broadcast SLA, `driver_disconnected`, `ride_not_active` error, `ride.events` Kafka consumption for session open/close, track archive
- [x] 6.3 Add Module: Geo-Indexing — Redis operations table (`GEOADD`, `ZREM`, `GEORADIUS`, `GEODIST`), `GET /internal/location/nearby-drivers` API contract, MongoDB `$nearSphere` fallback
- [x] 6.4 Add Module: User Pairing Location — `pairingLocationUpdate` Socket.io event + `UserPairingLocationStatus` upsert
- [x] 6.5 Add MongoDB schemas: `driverlocationstatuses`, `driverlocationtracks`, `tempdrivertrackings`, `userpairinglocationstatuses` (with index tables and TTL note)
- [x] 6.6 Add Redis Key Reference table confirming `driver:geo` is the only Redis key in the entire platform

## 7. PRD — Notification Service

- [x] 7.1 Create `specs/prd-notification-service/spec.md` with Module: Push Notification Dispatch — `push_notification` Kafka event → FCM (field constraints, dead-letter on max retries, invalid token handling, 4KB payload strip)
- [x] 7.2 Add ride event fan-out table (which events trigger push to which party, missing token handling)
- [x] 7.3 Add Module: Email Dispatch — `email_notification` Kafka event → SMTP (template fallback, SMTP failure retry), password reset flow (forgot-password API → `password_reset_requested` event → send email → reset-password API, no-enumeration rule, token expiry / reuse handling)
- [x] 7.4 Add Module: Service Health — `/health` with FCM + SMTP initialisation status, 503 degraded state
- [x] 7.5 Add Kafka Consumer Summary table

## 8. PRD — Analytics Service

- [x] 8.1 Create `specs/prd-analytics-service/spec.md` with Module: Admin Dashboard — Dashboard overview endpoint (response shape with `lagWarning`, MongoDB `secondaryPreferred`), Ride History paginated endpoint, Driver Search Log summary endpoint
- [x] 8.2 Add legacy HTML dashboard feature (static file serving, admin auth required, 404 on missing file)
- [x] 8.3 Add Module: Scheduled Cron Jobs — Daily Ride Summary (metrics list, zero-day handling, MongoDB lock deduplication), Driver Search Log Archival (backup-then-delete safety, rollback on failure), Full Collection Backup (per-collection failure isolation, disk-full alert)
- [x] 8.4 Add Module: Kafka Event Ingestion — `ride.events` fan-out updating local read copy (with idempotency note), `analytics.events` → `driversearchlogs` ingestion
- [x] 8.5 Add MongoDB schemas: `dailyridesummaries`, `driversearchlogbackups` (with index tables)

## 9. PRD Index

- [x] 9.1 Create `specs/prd-index/spec.md` — Service Registry table (all 6 services, ports, PRD file paths, stack, DB, Redis, Kafka role)
- [x] 9.2 Add Module and Feature Map — per-service tables mapping every module + feature to its file and section anchor (AI-navigable)
- [x] 9.3 Add API Endpoint Registry — flat table of every public endpoint (method, path, service, feature) with internal endpoints marked
- [x] 9.4 Add Kafka Topic Registry table
- [x] 9.5 Add MongoDB Collection Ownership table
- [x] 9.6 Add Redis Key Registry table

## 10. Cross-Service Dependency Map

- [x] 10.1 Create `specs/cross-service-dependency-map/spec.md` — dependency graph in Mermaid
- [x] 10.2 Add Feature-Level Dependency Matrix table (feature, primary service, depends-on, type, blocking flag)
- [x] 10.3 Add detailed narrative for each blocking cross-service dependency: driver assignment → Location Service, live tracking → Ride Service events, Ride Service → User Service read copy (fcmToken), Notification → User Service (password reset), Analytics → Ride Service (read copy), station proximity → stations data
- [x] 10.4 Add Service Boot-Up Order list for local development
- [x] 10.5 Add "safe to build in parallel" table and "requires cross-service coordination" table

## 11. Roadmap

- [x] 11.1 Create `specs/roadmap/spec.md` — Team assignments table (SE-1: Ride Service, SE-2: Location Service, SE-3: User Service + Gateway, IN-1: Notification Service, IN-2: Analytics Service)
- [x] 11.2 Add Phase 0 — Foundation (Week 1): docker-compose infra, shared parent POM, shared error library, Kafka topic creation, local config templates
- [x] 11.3 Add Phase 1 — Core Identity and Routing (Week 2–3): API Gateway full build, User Service Authentication module
- [x] 11.4 Add Phase 2 — User Profile + Location Foundation (Week 3–4): User Service Profile module, Location Service driver tracking + geo-index + nearby-drivers API
- [x] 11.5 Add Phase 3 — Ride Lifecycle (Week 4–6): Ride Service full build (all modules), Location Service tracking sessions (consuming `ride.events`)
- [x] 11.6 Add Phase 4 — Notifications + Analytics (Week 5–7): Notification Service full build (IN-1), Analytics Service full build (IN-2); runs in parallel with Phase 3 tail
- [x] 11.7 Add Phase 5 — Hardening (Week 7–8): end-to-end integration tests, MongoDB index verification, Kafka lag check, circuit breaker tests, security review
- [x] 11.8 Add Milestone Summary table (M0–M5 with target weeks and exit criteria)
- [x] 11.9 Add Risk Register table (5 risks with probability, impact, mitigation)
