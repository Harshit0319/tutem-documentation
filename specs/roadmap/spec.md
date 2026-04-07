## ADDED Requirements

---

# TUTEM Platform — Delivery Roadmap

**Team:** 2–3 senior engineers (SE), 2 interns (IN)
**Methodology:** Services are built in dependency order. Features within a service are built in the order they unblock other services. Interns own services with well-defined external interfaces and minimal cross-service logic.

---

## Team Assignments


| Role                     | Count            | Owns                                           |
| ------------------------ | ---------------- | ---------------------------------------------- |
| Senior Engineer 1 (SE-1) | 1                | Ride Service (lead), Cross-service integration |
| Senior Engineer 2 (SE-2) | 1                | Location Service (lead), Infrastructure setup  |
| Senior Engineer 3 (SE-3) | 1 (if available) | User Service, API Gateway                      |
| Intern 1 (IN-1)          | 1                | Notification Service                           |
| Intern 2 (IN-2)          | 1                | Analytics Service                              |


If only 2 senior engineers: SE-3 tasks are absorbed by SE-1 and SE-2.

---

## Phase 0 — Foundation (All team, Week 1)

**Goal:** Everyone can run the full stack locally. Standards are agreed and shared.


| Task                                                                 | Owner        | Output                                                                                                                     |
| -------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------- |
| Review and sign off on API protocol standard                         | All          | Shared understanding of error contract, status codes, pagination                                                           |
| Review and sign off on folder structure standard                     | All          | Shared understanding of hexagonal layout                                                                                   |
| Set up local infrastructure: MongoDB, Kafka, Redis                   | SE-2         | `docker-compose.yml` with all infra services                                                                               |
| Create shared Spring Boot parent POM / Gradle convention plugin      | SE-1 or SE-3 | Common dependency versions, plugin config reused across all services                                                       |
| Create shared error response library (error codes, `ApiError` class) | SE-1 or SE-3 | Shared JAR imported by all services                                                                                        |
| Create `application-local.yml` templates for all services            | SE-2         | Local config files per service                                                                                             |
| Confirm Kafka topics are created with correct partition counts       | SE-2         | Topics: `location.updates` (12), `ride.events` (6), `user.events` (3), `notification.dispatch` (3), `analytics.events` (6) |


**Exit criteria:** Any engineer can clone a service repo and run `./gradlew bootRun --args='--spring.profiles.active=local'` successfully.

---

## Phase 1 — Core Identity and Routing (Week 2–3)

**Goal:** API Gateway is live. Users can register, authenticate, and get a JWT. All traffic routes correctly.

### API Gateway (SE-3 or SE-1)


| Task                                         | Dependency |
| -------------------------------------------- | ---------- |
| Scaffold service with Spring Cloud Gateway   | Phase 0    |
| Implement path-based proxy route table       | Phase 0    |
| Implement CORS filter (`CorsConfig.java`)    | Phase 0    |
| Implement in-memory rate limiting filter     | Phase 0    |
| Implement `/health` endpoint                 | Phase 0    |
| Implement `/internal/`** block rule          | Phase 0    |
| Implement 404 and 502/504 error handlers     | Phase 0    |
| Write integration tests for routing and CORS | Above      |


### User Service — Authentication Module (SE-3 or SE-2)


| Task                                                                | Dependency        |
| ------------------------------------------------------------------- | ----------------- |
| Scaffold service (hexagonal structure, MongoDB config)              | Phase 0           |
| Implement `User` domain entity and `users` MongoDB schema           | Phase 0           |
| Implement `POST /api/auth/register`                                 | Above             |
| Implement `POST /api/auth/send-otp` + OTP rate limiting (in-memory) | Above             |
| Implement `POST /api/auth/verify-otp` + JWT issuance                | Above             |
| Implement `POST /api/auth/set-mpin` + `POST /api/auth/mpin-login`   | Above             |
| Implement `POST /api/auth/refresh-token`                            | JWT issuance done |
| Implement `POST /api/auth/admin-login`                              | Above             |
| Publish `user_registered` to `user.events` Kafka topic              | Kafka config done |
| Write unit tests for domain + service layer                         | Above             |


**Exit criteria (Phase 1):** A new user can register, verify OTP, receive JWT, and make an authenticated request through the API Gateway to User Service.

---

## Phase 2 — User Profile + Driver Location Foundation (Week 3–4)

**Goal:** User profiles are complete. Location Service can receive driver GPS and maintain the geo-index — unblocking Ride Service assignment in Phase 3.

### User Service — Profile Module (SE-3 or SE-2)


| Task                                                        | Dependency        |
| ----------------------------------------------------------- | ----------------- |
| Implement `GET /api/user/profile` + `PUT /api/user/profile` | Phase 1 User auth |
| Implement `POST /api/user/device-token`                     | Above             |
| Implement `POST /api/user/shuttle-optin` / `shuttle-optout` | Above             |
| Publish `device_token_updated` to `user.events`             | Kafka config done |
| Write integration tests for profile endpoints               | Above             |


### Location Service — Driver Tracking + Geo-Index (SE-2, lead)


| Task                                                                             | Dependency        |
| -------------------------------------------------------------------------------- | ----------------- |
| Scaffold service (hexagonal structure, MongoDB + Redis config)                   | Phase 0           |
| Implement Socket.io server (driver connect/disconnect lifecycle)                 | Phase 0           |
| Implement `clientEvent` handler — validate coords, upsert `DriverLocationStatus` | Socket.io done    |
| Implement `GEOADD` / `ZREM` Redis geo-index on every location update             | Redis config done |
| Implement insert `DriverLocationTrack` + `TempDriverTracking` (TTL)              | MongoDB done      |
| Publish `driver_location_updated` to `location.updates` Kafka topic              | Kafka done        |
| Implement `GET /internal/location/nearby-drivers` endpoint                       | Geo-index done    |
| Implement MongoDB `$nearSphere` fallback when Redis unavailable                  | Above             |
| Implement `cleanDriverLocationStatus` cron                                       | MongoDB done      |
| Write unit tests for geo-index logic + integration test for nearby-drivers API   | Above             |


**Exit criteria (Phase 2):** A driver can connect over Socket.io, send location updates, and `GET /internal/location/nearby-drivers` returns the correct sorted list.

---

## Phase 3 — Ride Lifecycle (Week 4–6)

**Goal:** Full ride lifecycle works end-to-end: book → assign → accept → complete. This is the most complex phase.

### Ride Service — Full Build (SE-1, lead)

**Module: Ride Lifecycle**


| Task                                                                             | Dependency                    |
| -------------------------------------------------------------------------------- | ----------------------------- |
| Scaffold service (hexagonal structure, MongoDB config)                           | Phase 0                       |
| Implement `RideRequest` domain entity + MongoDB schema                           | Phase 0                       |
| Implement `POST /api/ride-request/book-ride`                                     | Above                         |
| Implement `processRideRequests` cron — calls Location Service nearby-drivers API | Location Service Phase 2 done |
| Implement `PUT /api/ride-request/accept-ride` + `decline-ride`                   | Book ride done                |
| Implement `PUT /api/ride-request/driver-reached` + `start-trip`                  | Accept done                   |
| Implement `PUT /api/ride-request/complete-ride` + fare calculation               | Start trip done               |
| Implement `PUT /api/ride-request/cancel-ride`                                    | Accept done                   |
| Implement `rideMonitor` stale cleanup cron                                       | Book ride done                |
| Publish all `ride.events` Kafka events per state transition                      | Kafka done                    |
| Publish `notification.dispatch` events for driver + commuter FCM                 | Kafka done                    |


**Module: Driver Management**


| Task                                                         | Dependency           |
| ------------------------------------------------------------ | -------------------- |
| Implement `POST /api/driver-vehicle/register` + `PUT` update | Phase 0              |
| Implement driver search log writer                           | Assignment cron done |


**Module: Fare Management**


| Task                                                     | Dependency         |
| -------------------------------------------------------- | ------------------ |
| Implement `POST /api/fareamount` + `GET` admin endpoints | Phase 0            |
| Integrate fare calculation into `complete-ride`          | Complete ride done |


**Module: Station Queue**


| Task                                                             | Dependency |
| ---------------------------------------------------------------- | ---------- |
| Implement `POST /api/stations/checkin`                           | Phase 0    |
| Implement `POST /api/stations/driver-arrived` + FIFO match logic | Above      |


**Module: Pairing**


| Task                                       | Dependency |
| ------------------------------------------ | ---------- |
| Implement `POST /api/pairing-ride/request` | Phase 0    |
| Integrate `match_rider.py` subprocess call | Above      |
| Implement `DELETE /api/pairing-ride/:id`   | Above      |


**Location Service — Tracking Sessions (SE-2)**


| Task                                                                | Dependency                         |
| ------------------------------------------------------------------- | ---------------------------------- |
| Consume `ride.events` to open/close Socket.io tracking rooms        | Ride Service Kafka production done |
| Implement `joinTrackingRoom` / `leaveTrackingRoom` Socket.io events | Above                              |
| Implement `driver_disconnected` broadcast                           | Above                              |
| Write integration tests for tracking session lifecycle              | Above                              |


**Exit criteria (Phase 3):** Full ride flow works: commuter books → driver is auto-assigned by cron → driver accepts → trip completes → fare is calculated → all Kafka events are published.

---

## Phase 4 — Notifications + Analytics (Week 5–7, parallel with Phase 3 tail)

**Goal:** Notifications flow on all ride events. Admin dashboard shows live data. Interns own these services with senior review.

### Notification Service (IN-1, SE-1 reviews)


| Task                                                                              | Dependency                |
| --------------------------------------------------------------------------------- | ------------------------- |
| Scaffold service (hexagonal structure, FCM + SMTP config)                         | Phase 0                   |
| Implement Kafka consumer for `notification.dispatch` → `push_notification`        | Kafka + FCM done          |
| Implement Kafka consumer for `notification.dispatch` → `email_notification`       | Kafka + SMTP done         |
| Implement Kafka consumer for `ride.events` fan-out (assigned/completed/cancelled) | Ride Service Phase 3 done |
| Implement Kafka consumer for `user.events` → `password_reset_requested`           | User Service Phase 2 done |
| Implement dead-letter topic handling for failed FCM / SMTP                        | Above                     |
| Implement `/health` endpoint with FCM + SMTP status                               | Phase 0                   |
| Write unit tests for event routing logic                                          | Above                     |


### Analytics Service (IN-2, SE-2 reviews)


| Task                                                                                       | Dependency              |
| ------------------------------------------------------------------------------------------ | ----------------------- |
| Scaffold service (hexagonal structure, MongoDB config, read preference secondaryPreferred) | Phase 0                 |
| Implement Kafka consumer for `ride.events` → update local `riderequests` read copy         | Ride Service Kafka done |
| Implement Kafka consumer for `analytics.events` → populate `driversearchlogs` copy         | Ride Service Kafka done |
| Implement `GET /api/analytics/dashboard`                                                   | Read copy consumer done |
| Implement `GET /api/analytics/rides` (paginated, filtered)                                 | Read copy done          |
| Implement `GET /api/analytics/driver-search-logs`                                          | Read copy done          |
| Implement legacy HTML dashboard static file serving                                        | Phase 0                 |
| Implement `dailyRideSummary` cron + MongoDB lock deduplication                             | Read copy done          |
| Implement `archiveDriverSearchLogs` cron                                                   | Read copy done          |
| Implement `fullCollectionBackup` cron                                                      | Phase 0                 |
| Write unit tests for cron logic                                                            | Above                   |


**Exit criteria (Phase 4):** When a ride is completed, the driver and commuter receive FCM pushes within 1 second. The admin dashboard shows updated ride counts.

---

## Phase 5 — Hardening and Integration Testing (Week 7–8)

**Goal:** All services are production-ready. Full end-to-end flows are verified.


| Task                                                                                    | Owner        |
| --------------------------------------------------------------------------------------- | ------------ |
| End-to-end integration test: register → book → assign → complete → FCM push received    | SE-1 + SE-2  |
| Verify all MongoDB indexes are in place before load test                                | SE-2         |
| Verify Kafka consumer lag stays ≤ 1s under simulated load                               | SE-2         |
| Circuit breaker verification: kill FCM, verify Notification Service degrades gracefully | IN-1         |
| Verify gateway returns 502 on downstream failure, `/health` still 200                   | IN-2         |
| API Gateway rate limit smoke test                                                       | SE-3 or IN-1 |
| Security review: confirm no credentials in source, JWT per-service secrets, HTTPS only  | SE-1         |
| Update any PRD files where implementation diverged from spec                            | All          |


---

## Milestone Summary


| Milestone                      | Target Week | Exit Criteria                                                          |
| ------------------------------ | ----------- | ---------------------------------------------------------------------- |
| M0 — Foundation                | Week 1      | All engineers can run full stack locally                               |
| M1 — Auth + Routing            | Week 3      | User can register, get JWT, make authenticated request through gateway |
| M2 — Location Foundation       | Week 4      | Driver location updates, geo-index live, nearby-drivers API works      |
| M3 — Ride Lifecycle            | Week 6      | Full ride flow end-to-end with Kafka events flowing                    |
| M4 — Notifications + Analytics | Week 7      | FCM pushes on ride events; dashboard shows live data                   |
| M5 — Production Ready          | Week 8      | Integration tested, indexes verified, security reviewed                |


---

## Risk Register


| Risk                                                                | Probability | Impact | Mitigation                                                                                        |
| ------------------------------------------------------------------- | ----------- | ------ | ------------------------------------------------------------------------------------------------- |
| Location Service geo-index delayed — blocks Ride Service assignment | Medium      | High   | SE-2 prioritises `GET /internal/location/nearby-drivers` as first deliverable in Phase 2          |
| Intern unfamiliar with Kafka consumer patterns                      | Medium      | Medium | SE-1 pairs with interns in Week 1; provides a working Kafka consumer template from shared library |
| Python subprocess (`match_rider.py`) unstable                       | Medium      | Low    | Pairing feature is isolated in its own module; failure is logged without crashing the service     |
| MongoDB index missing at load time                                  | Medium      | High   | SE-2 owns index verification checklist in Phase 5                                                 |
| FCM / SMTP credentials not available for local dev                  | Low         | Medium | IN-1 uses FCM emulator and local SMTP (MailHog) in `application-local.yml`                        |


