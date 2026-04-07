## ADDED Requirements

---

# PRD Index — TUTEM Platform Knowledge Graph

This file is the single entry point for AI tooling and engineers to locate any module, feature, API endpoint, or Kafka event across all service PRDs. Load this file first; then load only the PRD file for the service you need.

---

## How to Use This Index

1. Find the service, module, or feature you need in the tables below
2. Note the **File** and **Section** columns
3. Load that file and navigate to that section
4. Do not load all PRD files simultaneously — each is self-contained

---

## Service Registry

| Service | Port | PRD File | Stack | DB | Redis | Kafka Role |
|---------|------|----------|-------|----|-------|------------|
| API Gateway | 5000 | `specs/prd-api-gateway/spec.md` | Java / Spring Cloud Gateway | None | None | None |
| User Service | 5001 | `specs/prd-user-service/spec.md` | Java / Spring Boot 3 | MongoDB | None | Producer |
| Ride Service | 5002 | `specs/prd-ride-service/spec.md` | Java / Spring Boot 3 | MongoDB | None | Producer + Consumer |
| Location Service | 5003 | `specs/prd-location-service/spec.md` | Java / Spring Boot 3 | MongoDB | Geo-index only | Producer |
| Notification Service | 5005 | `specs/prd-notification-service/spec.md` | Java / Spring Boot 3 | None (Phase 1) | None | Consumer |
| Analytics Service | 5006 | `specs/prd-analytics-service/spec.md` | Java / Spring Boot 3 | MongoDB | None | Consumer |

---

## Module and Feature Map

### API Gateway (`specs/prd-api-gateway/spec.md`)

| Module | Feature | Section |
|--------|---------|---------|
| Request Routing | Path-based proxy routing | `## Module: Request Routing` → `### Feature: Path-Based Proxy Routing` |
| Request Routing | Unmatched route handling | `### Feature: Unmatched Route Handling` |
| Request Routing | Upstream unavailable handling | `### Feature: Upstream Unavailable Handling` |
| CORS | CORS enforcement | `## Module: CORS` → `### Feature: CORS Enforcement` |
| Rate Limiting | Per-route rate limiting | `## Module: Rate Limiting` → `### Feature: Per-Route Rate Limiting` |
| Service Health | Gateway health check | `## Module: Service Health` → `### Feature: Gateway Health Check` |

---

### User Service (`specs/prd-user-service/spec.md`)

| Module | Feature | Section |
|--------|---------|---------|
| Authentication | User registration | `## Module: Authentication` → `### Feature: User Registration` |
| Authentication | OTP send and verify | `### Feature: OTP Send and Verify` |
| Authentication | MPIN login | `### Feature: MPIN Login` |
| Authentication | JWT refresh | `### Feature: JWT Refresh` |
| User Profile | Get and update profile | `## Module: User Profile` → `### Feature: Get and Update Profile` |
| User Profile | Device token registration | `### Feature: Device Token Registration` |
| User Profile | Shuttle opt-in | `### Feature: Shuttle Opt-in` |
| Admin | Admin login | `## Module: Admin` → `### Feature: Admin Login` |

---

### Ride Service (`specs/prd-ride-service/spec.md`)

| Module | Feature | Section |
|--------|---------|---------|
| Ride Lifecycle | Book a ride | `## Module: Ride Lifecycle` → `### Feature: Book a Ride` |
| Ride Lifecycle | Driver assignment (cron) | `### Feature: Driver Assignment (Cron)` |
| Ride Lifecycle | Driver accept / decline | `### Feature: Driver Accept / Decline Ride` |
| Ride Lifecycle | Ride status transitions | `### Feature: Ride Status Transitions` |
| Ride Lifecycle | Cancel ride | `### Feature: Cancel Ride` |
| Ride Lifecycle | Stale ride cleanup (cron) | `### Feature: Stale Ride Cleanup (Cron)` |
| Driver Management | Register / update vehicle | `## Module: Driver Management` → `### Feature: Register / Update Vehicle Details` |
| Driver Management | Driver search log | `### Feature: Driver Search Log` |
| Fare Management | Fare configuration (admin) | `## Module: Fare Management` → `### Feature: Fare Configuration (Admin)` |
| Fare Management | Fare calculation | `### Feature: Fare Calculation` |
| Station Queue | Station check-in | `## Module: Station Queue (FIFO)` → `### Feature: Station Check-In` |
| User-to-User Pairing | Create and match pairing | `## Module: User-to-User Pairing` → `### Feature: Create and Match Pairing Request` |

---

### Location Service (`specs/prd-location-service/spec.md`)

| Module | Feature | Section |
|--------|---------|---------|
| Driver Location Tracking | Receive and persist location update | `## Module: Driver Location Tracking` → `### Feature: Receive and Persist Driver Location Update` |
| Driver Location Tracking | Connect / disconnect lifecycle | `### Feature: Driver Connect / Disconnect Lifecycle` |
| Driver Location Tracking | Stale driver cleanup (cron) | `### Feature: Stale Driver Cleanup (Cron)` |
| Live Ride Tracking | Start live tracking session | `## Module: Live Ride Tracking` → `### Feature: Start Live Tracking Session` |
| Live Ride Tracking | Location track archive | `### Feature: Location Track Archive` |
| Geo-Indexing | Redis geo-index maintenance | `## Module: Geo-Indexing` → `### Feature: Redis Geo-Index Maintenance` |
| Geo-Indexing | Nearby drivers query API | `### Feature: Nearby Drivers Query API` |
| User Pairing Location | User pairing location update | `## Module: User Pairing Location` → `### Feature: User Pairing Location Update` |

---

### Notification Service (`specs/prd-notification-service/spec.md`)

| Module | Feature | Section |
|--------|---------|---------|
| Push Notification Dispatch | FCM push notification | `## Module: Push Notification Dispatch` → `### Feature: FCM Push Notification` |
| Push Notification Dispatch | Notification dispatch for ride events | `### Feature: Notification Dispatch for Ride Events` |
| Email Dispatch | Transactional email | `## Module: Email Dispatch` → `### Feature: Transactional Email (Ride Events)` |
| Email Dispatch | Password reset email | `### Feature: Password Reset Email` |
| Service Health | Health check | `## Module: Service Health` → `### Feature: Health Check` |

---

### Analytics Service (`specs/prd-analytics-service/spec.md`)

| Module | Feature | Section |
|--------|---------|---------|
| Admin Dashboard | Dashboard overview | `## Module: Admin Dashboard` → `### Feature: Dashboard Overview` |
| Admin Dashboard | Ride history (paginated) | `### Feature: Ride History (Paginated)` |
| Admin Dashboard | Driver search log summary | `### Feature: Driver Search Log Summary` |
| Admin Dashboard | Legacy admin dashboards | `### Feature: Legacy Admin Dashboard (Static HTML)` |
| Scheduled Cron Jobs | Daily ride summary | `## Module: Scheduled Cron Jobs` → `### Feature: Daily Ride Summary (Cron)` |
| Scheduled Cron Jobs | Driver search log archival | `### Feature: Driver Search Log Archival (Cron)` |
| Scheduled Cron Jobs | Full collection backup | `### Feature: Full Collection Backup (Cron)` |
| Kafka Event Ingestion | Ride event ingestion | `## Module: Kafka Event Ingestion` → `### Feature: Ride Event Ingestion` |
| Kafka Event Ingestion | Driver search event ingestion | `### Feature: Driver Search Event Ingestion` |

---

## API Endpoint Registry

All public endpoints (proxied through API Gateway). Internal endpoints are marked.

| Method | Path | Service | Feature |
|--------|------|---------|---------|
| `POST` | `/api/auth/register` | User Service | User Registration |
| `POST` | `/api/auth/send-otp` | User Service | OTP Send and Verify |
| `POST` | `/api/auth/verify-otp` | User Service | OTP Send and Verify |
| `POST` | `/api/auth/set-mpin` | User Service | MPIN Login |
| `POST` | `/api/auth/mpin-login` | User Service | MPIN Login |
| `POST` | `/api/auth/refresh-token` | User Service | JWT Refresh |
| `POST` | `/api/auth/admin-login` | User Service | Admin Login |
| `POST` | `/api/auth/forgot-password` | User Service → Notification | Password Reset Email |
| `POST` | `/api/auth/reset-password` | User Service → Notification | Password Reset Email |
| `GET` | `/api/user/profile` | User Service | Get and Update Profile |
| `PUT` | `/api/user/profile` | User Service | Get and Update Profile |
| `POST` | `/api/user/device-token` | User Service | Device Token Registration |
| `POST` | `/api/user/shuttle-optin` | User Service | Shuttle Opt-in |
| `POST` | `/api/user/shuttle-optout` | User Service | Shuttle Opt-in |
| `POST` | `/api/ride-request/book-ride` | Ride Service | Book a Ride |
| `PUT` | `/api/ride-request/accept-ride` | Ride Service | Driver Accept / Decline |
| `PUT` | `/api/ride-request/decline-ride` | Ride Service | Driver Accept / Decline |
| `PUT` | `/api/ride-request/driver-reached` | Ride Service | Ride Status Transitions |
| `PUT` | `/api/ride-request/start-trip` | Ride Service | Ride Status Transitions |
| `PUT` | `/api/ride-request/complete-ride` | Ride Service | Ride Status Transitions |
| `PUT` | `/api/ride-request/cancel-ride` | Ride Service | Cancel Ride |
| `POST` | `/api/driver-vehicle/register` | Ride Service | Register / Update Vehicle |
| `PUT` | `/api/driver-vehicle/:vehicleId` | Ride Service | Register / Update Vehicle |
| `POST` | `/api/fareamount` | Ride Service | Fare Configuration (Admin) |
| `GET` | `/api/fareamount` | Ride Service | Fare Configuration (Admin) |
| `POST` | `/api/stations/checkin` | Ride Service | Station Check-In |
| `POST` | `/api/stations/driver-arrived` | Ride Service | Station Check-In |
| `POST` | `/api/pairing-ride/request` | Ride Service | Create and Match Pairing |
| `DELETE` | `/api/pairing-ride/:pairingRequestId` | Ride Service | Create and Match Pairing |
| `GET` | `/api/analytics/dashboard` | Analytics Service | Dashboard Overview |
| `GET` | `/api/analytics/rides` | Analytics Service | Ride History (Paginated) |
| `GET` | `/api/analytics/driver-search-logs` | Analytics Service | Driver Search Log Summary |
| `GET` | `/health` | All services (local) | Health Check |
| `GET` | `/internal/location/nearby-drivers` | Location Service | Nearby Drivers Query (**internal only — blocked at gateway**) |

---

## Kafka Topic Registry

| Topic | Partitions | Key | Producers | Consumers |
|-------|-----------|-----|-----------|-----------|
| `location.updates` | 12 | `driverId` | Location Service | Ride Service, Analytics Service |
| `ride.events` | 6 | `rideRequestId` | Ride Service | Notification Service, Analytics Service |
| `user.events` | 3 | `userId` | User Service | Notification Service, Analytics Service |
| `notification.dispatch` | 3 | `targetUserId` | Ride Service, User Service | Notification Service |
| `analytics.events` | 6 | `eventType` | Ride Service | Analytics Service |

---

## MongoDB Collection Ownership

| Collection | Primary Owner | Phase 1 Read Copies |
|------------|---------------|---------------------|
| `users` | User Service | Ride Service |
| `riderequests` | Ride Service | Analytics Service |
| `userpairingriderequests` | Ride Service | — |
| `drivervehicledetails` | Ride Service | — |
| `stations` | Ride Service | — |
| `fifostationdata` | Ride Service | — |
| `fares` | Ride Service | — |
| `driversearchlogs` | Ride Service | Analytics Service |
| `driverlocationstatuses` | Location Service | — |
| `driverlocationtracks` | Location Service | — |
| `tempdrivertrackings` | Location Service | — |
| `userpairinglocationstatuses` | Location Service | — |
| `userlocationstatuses` | Location Service | — |
| `dailyridesummaries` | Analytics Service | — |
| `driversearchlogbackups` | Analytics Service | — |

---

## Redis Key Registry

| Key | Service | Operations | Purpose |
|-----|---------|------------|---------|
| `driver:geo` | Location Service | `GEOADD`, `ZREM`, `GEORADIUS`, `GEODIST` | Driver geo-index for assignment queries |

**All other services have zero Redis dependency.**
