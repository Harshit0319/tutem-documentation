## ADDED Requirements

---

# PRD: Analytics Service

**Service:** `analytics-service`
**Port:** 5006
**Stack:** Java 17 / Spring Boot 3
**Database:** MongoDB (dedicated `analytics-service` database, read preference `secondaryPreferred`)
**Redis:** None
**Kafka:** Consumes from `ride.events`, `analytics.events`, `location.updates`
**Owned collections:** `dailyridesummaries`, `driversearchlogbackups`
**Phase 1 read copies:** `riderequests` (from Ride Service), `driversearchlogs` (from Ride Service)

---

## Module: Admin Dashboard

Serves authenticated REST endpoints for operational data and ride statistics. All endpoints require a JWT with `role: admin`.

---

### Feature: Dashboard Overview

Returns aggregated platform-level metrics for the admin dashboard.

**API Contract:**

| | |
|--|--|
| Method + Path | `GET /api/analytics/dashboard` |
| Auth | JWT required (admin role) |
| Query params | `from=<ISO8601>&to=<ISO8601>` (optional; defaults to last 30 days) |
| 200 | Dashboard summary object (see below) |
| 403 | `{ "error": "insufficient_permissions" }` |

**Response (200):**
```json
{
  "period": {
    "from": "2025-02-10T00:00:00.000Z",
    "to": "2025-03-10T23:59:59.000Z"
  },
  "rides": {
    "total": 1240,
    "completed": 1080,
    "cancelled": 160,
    "completionRate": 0.87
  },
  "fare": {
    "totalCollected": 148500,
    "avgFare": 137.5
  },
  "users": {
    "totalRegistered": 4820,
    "activeCommuters": 1230,
    "activeDrivers": 340
  },
  "drivers": {
    "totalOnline": 87,
    "avgSearchRadius": 2400
  },
  "lagWarning": null
}
```

`lagWarning` is populated if MongoDB secondary lag exceeds 5 seconds: `{ "lagSeconds": 7, "note": "data may be slightly stale" }`.

MongoDB read preference: `secondaryPreferred` for all dashboard queries.

#### Scenario: Admin fetches dashboard
- **WHEN** an authenticated admin calls GET /api/analytics/dashboard
- **THEN** HTTP 200 is returned with aggregated metrics for the requested period; data is read from MongoDB secondary

#### Scenario: Non-admin JWT
- **WHEN** a JWT without `role: admin` is used
- **THEN** HTTP 403 is returned with `{ "error": "insufficient_permissions" }`

#### Scenario: MongoDB secondary lagging
- **WHEN** MongoDB secondary lag exceeds 5 seconds at query time
- **THEN** the response still returns HTTP 200 with the latest committed data; `lagWarning` is populated with the lag duration

---

### Feature: Ride History (Paginated)

Returns a paginated list of ride requests for admin review.

**API Contract:**

| | |
|--|--|
| Method + Path | `GET /api/analytics/rides` |
| Auth | JWT required (admin role) |
| Query params | `page=<int>&limit=<int>&status=<string>&from=<ISO8601>&to=<ISO8601>&userId=<string>&driverId=<string>` |
| 200 | Paginated ride list (see pagination standard) |
| 403 | `{ "error": "insufficient_permissions" }` |

**Response (200):**
```json
{
  "data": [
    {
      "rideRequestId": "64a1f3e8c2a4b20017e4f502",
      "userId": "64a1f3e8c2a4b20017e4f503",
      "driverId": "64a1f3e8c2a4b20017e4f501",
      "status": "endtrip",
      "fareAmountCalculated": 135,
      "FinalTraveldistance": 12.8,
      "createdAt": "2025-03-10T14:23:00.000Z",
      "completedAt": "2025-03-10T14:55:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1080,
    "hasNext": true
  }
}
```

Query uses indexed fields (`createdAt`, `status`, `userId`, `driverId`) on the Phase 1 read copy of `riderequests`. Response time SHALL be ≤ 500ms at p95.

#### Scenario: Paginated ride list returned
- **WHEN** an admin calls GET /api/analytics/rides with optional filters
- **THEN** HTTP 200 is returned with a paginated result set; results are sorted by `createdAt` descending

#### Scenario: Large result set — index used
- **WHEN** the `riderequests` read copy has >1M documents
- **THEN** the query uses indexed fields; response time remains ≤ 500ms p95

---

### Feature: Driver Search Log Summary

Returns aggregated stats on driver search attempts.

**API Contract:**

| | |
|--|--|
| Method + Path | `GET /api/analytics/driver-search-logs` |
| Auth | JWT required (admin role) |
| Query params | `from=<ISO8601>&to=<ISO8601>` |
| 200 | Aggregated summary object |
| 403 | `{ "error": "insufficient_permissions" }` |

**Response (200):**
```json
{
  "period": {
    "from": "2025-03-01T00:00:00.000Z",
    "to": "2025-03-10T23:59:59.000Z"
  },
  "totalSearches": 3420,
  "uniqueDriversSearched": 87,
  "assignmentConversionRate": 0.76,
  "noDriverFoundCount": 821
}
```

#### Scenario: No logs in time range
- **WHEN** no `DriverSearchLog` records exist for the requested range
- **THEN** HTTP 200 is returned with all counts as 0; not HTTP 404

---

### Feature: Legacy Admin Dashboard (Static HTML)

Serves static HTML admin dashboards at legacy URL paths for backward compatibility.

**API Contracts:**

| Path | Auth | Response |
|------|------|----------|
| `GET /iitb-dashboard` | JWT required (admin) | `200 text/html` — HTML file from `public/` |
| `GET /bitshyd-dashboard` | JWT required (admin) | `200 text/html` |
| `GET /srinagaradmin` | JWT required (admin) | `200 text/html` |
| `GET /admin` | JWT required (admin) | `200 text/html` |
| `GET /rides` | JWT required (admin) | `200 text/html` |
| `GET /driverTrack` | JWT required (admin) | `200 text/html` |

Static files served from `public/` directory. All legacy paths require admin authentication; unauthenticated requests return HTTP 401.

#### Scenario: Dashboard file served
- **WHEN** an authenticated admin requests a legacy dashboard path
- **THEN** the corresponding HTML file is served from `public/` with `Content-Type: text/html`

#### Scenario: Dashboard file missing
- **WHEN** the HTML file is not found in `public/`
- **THEN** HTTP 404 is returned; the service does not crash; a missing-asset warning is logged

#### Scenario: Unauthenticated request
- **WHEN** a request arrives at a dashboard path without a valid admin JWT
- **THEN** HTTP 401 is returned; the HTML file is not served

---

## Module: Scheduled Cron Jobs

All cron jobs run on a configurable schedule. In Phase 1 with multiple replicas, cron deduplication is achieved via MongoDB atomic operations (`findOneAndUpdate` with a `lockedAt` TTL field) since Redis is not available in this service.

---

### Feature: Daily Ride Summary (Cron)

Computes and persists a daily aggregate of ride metrics.

**Cron:** `dailyRideSummary` — runs once per day at configured time (e.g. 00:05 UTC).

**No direct HTTP endpoint.**

**Written to:** `dailyridesummaries` collection.

**Computed metrics per day:**
- `totalRides` — count of all ride requests created that day
- `completedRides` — count with `status: "endtrip"`
- `cancelledRides` — count with `status: "cancelledbyuser"` or `"cancelledbydriver"`
- `totalFareCollected` — sum of `fareAmountCalculated` for completed rides
- `avgFare` — `totalFareCollected / completedRides`
- `totalDrivers` — distinct driver IDs with at least one ride that day
- `totalUsers` — distinct user IDs with at least one ride that day

**Kafka Event Consumed — `analytics.events` → `ride_summary`:**
```json
{
  "eventType": "ride_summary",
  "rideRequestId": "64a1f3e8c2a4b20017e4f502",
  "userId": "64a1f3e8c2a4b20017e4f503",
  "driverId": "64a1f3e8c2a4b20017e4f501",
  "fareAmount": 120,
  "fareAmountCalculated": 135,
  "distance": 12.8,
  "duration": 1620,
  "status": "endtrip",
  "organization": "Personal",
  "occurredAt": "2025-03-10T14:55:00.000Z"
}
```

#### Scenario: Daily summary generated successfully
- **WHEN** the cron fires for the previous calendar day
- **THEN** a `DailyRideSummary` document is persisted; the cron logs `{ event: "cron_run", name: "dailyRideSummary", status: "success", recordsProcessed: N }`

#### Scenario: Zero rides on a day
- **WHEN** no rides were completed in the previous 24 hours
- **THEN** a summary document with all zero counts is still persisted; the cron does not skip

#### Scenario: Duplicate cron run prevented
- **WHEN** multiple Analytics Service replicas are running
- **THEN** a MongoDB `findOneAndUpdate` with `lockedAt` TTL ensures only one replica executes the summary write; the second replica detects the lock and exits without writing

---

### Feature: Driver Search Log Archival (Cron)

Moves aged `DriverSearchLog` entries to `DriversearchLogBackup` and purges the primary collection.

**Cron:** `archiveDriverSearchLogs` — runs on configurable interval (e.g. daily at 01:00 UTC).

**No direct HTTP endpoint.**

**Operation:**
1. Query `driversearchlogs` read copy for entries older than the configured threshold
2. Bulk-insert into `driversearchlogbackups` with `backedUpAt: now`
3. Only after successful backup write: delete the source records
4. Log `{ event: "archive_complete", count: N, backedUpAt }`

#### Scenario: Archival completes successfully
- **WHEN** the cron runs and aged records exist
- **THEN** records are inserted into `driversearchlogbackups`; source records are deleted only after backup write confirms; count is logged

#### Scenario: Backup write fails mid-operation
- **WHEN** the bulk insert into `driversearchlogbackups` fails
- **THEN** the delete step is skipped; source records are preserved; the error is logged; the cron retries on the next cycle

#### Scenario: No aged records
- **WHEN** no records exist beyond the configured threshold
- **THEN** the cron exits cleanly with `{ count: 0 }`; no error is thrown

---

### Feature: Full Collection Backup (Cron)

Performs periodic snapshots of configured collections for disaster recovery.

**Cron:** `fullCollectionBackup` — runs on configured schedule (e.g. weekly).

**No direct HTTP endpoint.**

#### Scenario: Backup completes for all collections
- **WHEN** the cron runs
- **THEN** each configured collection is snapshotted; completion is logged with `{ event: "backup_complete", collections: [...], timestamp }`

#### Scenario: One collection backup fails
- **WHEN** a single collection backup write fails
- **THEN** the error is logged with the collection name; backup continues for remaining collections; the cron does not abort entirely

#### Scenario: Disk space insufficient
- **WHEN** available disk space is below the minimum threshold
- **THEN** a critical error is logged `{ error: "backup_disk_full" }`; an alert is dispatched to Platform Admin via Notification Service; no backup is attempted

---

## Module: Kafka Event Ingestion

Consumes ride and analytics events to keep read copies up to date and feed dashboard aggregations.

---

### Feature: Ride Event Ingestion

Consumes `ride.events` to update the local `riderequests` read copy in near-real time.

**Kafka Events Consumed — `ride.events`:**

| `eventType` | Action |
|-------------|--------|
| `ride_created` | Insert new document into local `riderequests` read copy |
| `ride_assigned` | Update `status`, `driverId`, `expectedReachTime` on local copy |
| `ride_completed` | Update `status`, `fareAmountCalculated`, `FinalTraveldistance`, `completedAt` |
| `ride_cancelled` | Update `status`, `cancelledAt`, `reasonForCancelling` |

#### Scenario: Ride event updates local read copy
- **WHEN** a `ride_completed` event is consumed
- **THEN** the corresponding document in the local `riderequests` read copy is updated; the dashboard aggregate reflects the change on the next query

#### Scenario: Duplicate event received (at-least-once delivery)
- **WHEN** the same `ride_completed` event is delivered twice
- **THEN** the `findOneAndUpdate` with the rideRequestId is idempotent; no duplicate record is created

---

### Feature: Driver Search Event Ingestion

Consumes `analytics.events` → `driver_search_completed` to populate the `driversearchlogs` read copy.

**Kafka Event Consumed — `analytics.events`:**
```json
{
  "eventType": "driver_search_completed",
  "userId": "64a1f3e8c2a4b20017e4f503",
  "socketId": "ABC123",
  "searchLocation": {
    "latitude": 17.5449,
    "longitude": 78.5719,
    "destinationLat": 17.4375,
    "destinationLng": 78.4483
  },
  "radiusUsed": 2000,
  "maxRadius": 5000,
  "driversFoundCount": 3,
  "status": "success",
  "occurredAt": "2025-03-10T14:23:00.000Z"
}
```

#### Scenario: Search log record inserted
- **WHEN** a `driver_search_completed` event is consumed
- **THEN** a `DriverSearchLog` record is inserted into the local read copy for dashboard aggregation

---

## MongoDB Schemas

### `dailyridesummaries` collection

| Field | Type | Required | Indexed | Description |
|-------|------|----------|---------|-------------|
| `_id` | ObjectId | yes | PK | |
| `date` | date | yes | yes (unique) | Calendar date of the summary |
| `totalRides` | number | yes | no | |
| `completedRides` | number | yes | no | |
| `cancelledRides` | number | yes | no | |
| `totalFareCollected` | number | yes | no | Sum of `fareAmountCalculated` |
| `avgFare` | number | yes | no | |
| `totalDrivers` | number | yes | no | Distinct drivers with rides that day |
| `totalUsers` | number | yes | no | Distinct users with rides that day |
| `createdAt` | date | yes | no | |

**Indexes:** `date` (unique)

---

### `driversearchlogbackups` collection

| Field | Type | Required | Indexed | Description |
|-------|------|----------|---------|-------------|
| `_id` | ObjectId | yes | PK | |
| *(all fields from `driversearchlogs`)* | — | — | — | Full document copy |
| `backedUpAt` | date | yes | yes | Timestamp when archived |
| `createdAt` | date | yes | yes | Original log creation time |

**Indexes:** `backedUpAt`, `createdAt`
