## ADDED Requirements

---

# PRD: Location Service

**Service:** `location-service`
**Port:** 5003
**Stack:** Java 17 / Spring Boot 3
**Database:** MongoDB (dedicated `location-service` database)
**Redis:** Yes — geo-indexing ONLY (`GEOADD` / `GEORADIUS` / `GEODIST` on `driver:geo` key). No caching, no rate limiting, no Socket.io adapter, no Redlock.
**Real-time transport:** Socket.io (in-process, single-instance in Phase 1; in-memory adapter)
**Kafka:** Produces to `location.updates`; Consumes from `ride.events` (to start/stop tracking sessions)
**Owned collections:** `driverlocationstatuses`, `driverlocationtracks`, `tempdrivertrackings`, `userpairinglocationstatuses`, `userlocationstatuses`

---

## Module: Driver Location Tracking

Handles real-time driver GPS updates received over Socket.io, persists them to MongoDB, and maintains the Redis geo-index used by Ride Service for driver assignment queries.

---

### Feature: Receive and Persist Driver Location Update

Drivers emit their GPS position over a persistent Socket.io connection. Each update is persisted and indexed.

**Socket.io event (inbound — from driver client):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Event name | `clientEvent` | — | Emitted by driver app |
| `driverId` | string | yes | Driver's user ID |
| `lat` | number | yes | Latitude |
| `lng` | number | yes | Longitude |
| `accuracy` | number | no | GPS accuracy in metres |
| `status` | string | yes | `active` / `idle` / `assigned` / `pending` |
| `rideId` | string | no | Active ride request ID; null when idle |

**On receipt, the service SHALL:**
1. Validate coordinates (lat ∈ [-90, 90], lng ∈ [-180, 180])
2. Upsert `DriverLocationStatus` in MongoDB (last known position)
3. Insert a `DriverLocationTrack` record (full history)
4. Insert a `TempDriverTracking` record (TTL: 600 s)
5. `GEOADD driver:geo <lng> <lat> <driverId>` in Redis
6. Publish a `driver_location_updated` event to Kafka `location.updates`

**Kafka Event Produced — `location.updates`:**
```json
{
  "eventType": "driver_location_updated",
  "driverId": "64a1f3e8c2a4b20017e4f501",
  "rideId": "64a1f3e8c2a4b20017e4f502",
  "location": {
    "type": "Point",
    "coordinates": [78.5719, 17.5449]
  },
  "accuracy": 12.5,
  "status": "assigned",
  "occurredAt": "2025-03-10T14:23:00.000Z"
}
```
Topic key: `driverId`. Partition count: 12 (high-frequency topic).

#### Scenario: Valid location update received
- **WHEN** a driver emits a `clientEvent` with valid coordinates
- **THEN** `DriverLocationStatus` is upserted; a `DriverLocationTrack` record is inserted; Redis geo-index is updated; a `driver_location_updated` Kafka event is published

#### Scenario: Coordinates outside valid bounds
- **WHEN** coordinates are outside valid lat/lng bounds
- **THEN** the update is rejected and logged; the previous valid location is preserved in MongoDB and Redis; no Kafka event is emitted

#### Scenario: Driver emits faster than throttle rate
- **WHEN** a driver emits updates faster than the configured throttle (e.g. >1 per second)
- **THEN** updates are processed at the throttle rate; excess events are dropped, not queued

#### Scenario: Driver socket disconnects mid-update
- **WHEN** the socket disconnects before an update write completes
- **THEN** the partial update is discarded; the last fully committed location remains in Redis and MongoDB

---

### Feature: Driver Connect / Disconnect Lifecycle

Manages driver online/offline status when their Socket.io connection is established or dropped.

**Socket.io events:**

| Event | Direction | Payload | Action |
|-------|-----------|---------|--------|
| `connection` | inbound | socket handshake with `driverId` query param | Upsert `DriverLocationStatus` with `status: "active"`, set `loginTime` |
| `disconnect` | inbound | *(automatic)* | Upsert `DriverLocationStatus` with `status: "inactive"`, set `logoutTime`; remove driver from Redis geo-index via `ZREM driver:geo <driverId>` |

#### Scenario: Driver connects
- **WHEN** a driver establishes a Socket.io connection with a valid `driverId`
- **THEN** `DriverLocationStatus` is upserted with `status: "active"` and `loginTime: now`

#### Scenario: Driver disconnects
- **WHEN** a driver's socket connection is dropped (gracefully or due to network loss)
- **THEN** `DriverLocationStatus` is updated with `status: "inactive"` and `logoutTime: now`; the driver's entry is removed from the Redis geo-index so they are not returned in assignment queries

---

### Feature: Stale Driver Cleanup (Cron)

A scheduled cron removes `DriverLocationStatus` entries that have not been updated within the configured offline threshold.

**No direct HTTP endpoint — triggered by cron only.**

**Cron:** `cleanDriverLocationStatus` — runs on a configurable interval.

#### Scenario: Stale entry cleaned up
- **WHEN** a `DriverLocationStatus` entry has not been updated beyond the offline threshold
- **THEN** the document is deleted from MongoDB; `ZREM driver:geo <driverId>` is called to remove from Redis geo-index

#### Scenario: Driver reconnects just as cron fires
- **WHEN** a driver sends a location update at the same moment the cleanup cron targets their entry
- **THEN** last-write-wins semantics apply; if the location update commits first, the cron skips that entry

---

## Module: Live Ride Tracking

Manages real-time location sharing between a driver and a commuter during an active ride.

---

### Feature: Start Live Tracking Session

When a ride transitions to `assigned` or `in-progress`, the commuter joins a Socket.io room to receive live driver coordinates.

**Socket.io events:**

| Event | Direction | Payload | Description |
|-------|-----------|---------|-------------|
| `joinTrackingRoom` | inbound (from commuter) | `{ "rideId": string, "userId": string }` | Commuter joins the driver's tracking room |
| `leaveTrackingRoom` | inbound (from commuter) | `{ "rideId": string }` | Commuter leaves the room |
| `locationUpdate` | outbound (to room) | `{ "lat": number, "lng": number, "accuracy": number, "timestamp": string }` | Broadcast to all room members on each driver `clientEvent` |
| `driver_disconnected` | outbound (to room) | `{ "rideId": string }` | Emitted when driver socket drops during active ride |
| `ride_not_active` | outbound (to commuter) | `{ "error": "ride_not_active" }` | Emitted if commuter tries to join a completed/cancelled ride room |

**Cross-service dependency:** Location Service consumes `ride.events` from Kafka to know when rides transition to `assigned` (start tracking) and `endtrip` / `cancelledbyuser` / `cancelledbydriver` (stop tracking and close the room).

#### Scenario: Commuter joins active ride room
- **WHEN** a commuter emits `joinTrackingRoom` for an active ride
- **THEN** the socket is added to the ride's room; all subsequent `locationUpdate` events from the driver are broadcast to the room within 200ms

#### Scenario: Commuter joins completed ride room
- **WHEN** a commuter emits `joinTrackingRoom` for a ride that is already in `endtrip` or `cancelled` state
- **THEN** a `ride_not_active` socket error event is returned; no room join occurs

#### Scenario: Driver disconnects during active ride
- **WHEN** the driver's socket drops while the ride is `ontrip`
- **THEN** a `driver_disconnected` event is broadcast to all room members; the tracking session is flagged as interrupted in MongoDB

---

### Feature: Location Track Archive

Full GPS path for each active ride is stored in `DriverLocationTrack` for post-trip review and admin analytics.

**No direct HTTP endpoint — records are written on every inbound driver `clientEvent`.**

#### Scenario: Track record written on each update
- **WHEN** a driver emits a location update while on an active ride (`rideId` is set)
- **THEN** a `DriverLocationTrack` document is inserted with `driverId`, `rideId`, coordinates, accuracy, and timestamp

---

## Module: Geo-Indexing

Manages the Redis geo-index (`driver:geo`) used by Ride Service to find the nearest available driver. This is the only Redis usage in the entire platform.

---

### Feature: Redis Geo-Index Maintenance

The geo-index is kept in sync with live driver positions.

**Redis operations used:**

| Operation | When | Purpose |
|-----------|------|---------|
| `GEOADD driver:geo <lng> <lat> <driverId>` | On every valid location update | Update driver position in index |
| `ZREM driver:geo <driverId>` | On disconnect or stale cleanup | Remove offline driver from index |

**Read operations** (performed by Ride Service via Location Service HTTP API — see below):

| Operation | Description |
|-----------|-------------|
| `GEORADIUS driver:geo <lng> <lat> <radius> m ASC COUNT 10` | Find nearest drivers within radius |
| `GEODIST driver:geo <driverId> <stationKey> m` | Check driver proximity to a station |

---

### Feature: Nearby Drivers Query API

Exposes an internal HTTP endpoint so Ride Service can query the Redis geo-index without direct Redis access.

**API Contract:**

| | |
|--|--|
| Method + Path | `GET /internal/location/nearby-drivers` |
| Auth | Internal service header (`X-Internal-Service: ride-service`) |
| Query params | `lat=<number>&lng=<number>&radius=<metres>&limit=<number>` |
| 200 | `{ "success": true, "message": "nearby drivers retrieved", "data": [{ "driverId": string, "distanceMetres": number }] }` sorted ascending by distance |
| 400 | `{ "success": false, "message": "validation failed", "data": null, "errors": [...] }` |

This endpoint is **not** exposed through the API Gateway. It is called service-to-service only.

**Cross-service dependency:** Ride Service calls this endpoint during the `processRideRequests` cron to find candidate drivers for assignment.

#### Scenario: Drivers found within radius
- **WHEN** Ride Service calls GET /internal/location/nearby-drivers with valid coords and radius
- **THEN** HTTP 200 is returned with an array of driver IDs sorted by ascending distance

#### Scenario: No drivers in radius
- **WHEN** no active drivers exist within the requested radius
- **THEN** HTTP 200 is returned with `{ "success": true, "message": "no drivers found within the requested radius", "data": [] }`

#### Scenario: Redis geo-index unavailable
- **WHEN** Redis is temporarily unavailable
- **THEN** the endpoint falls back to a MongoDB `$nearSphere` query on the `2dsphere` index on `driverlocationstatuses.location`; HTTP 200 is returned; a warning is logged

---

## Module: User Pairing Location

Tracks location of users involved in user-to-user pairing rides.

---

### Feature: User Pairing Location Update

Users involved in a pairing ride share their location via Socket.io (similar to driver tracking but for peer-to-peer pairing sessions).

**Socket.io event (inbound — from commuter in pairing session):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Event name | `pairingLocationUpdate` | — | |
| `userId` | string | yes | |
| `pairingRequestId` | string | yes | |
| `lat` | number | yes | |
| `lng` | number | yes | |
| `accuracy` | number | no | |

On receipt: upsert `UserPairingLocationStatus` in MongoDB.

#### Scenario: Pairing location updated
- **WHEN** a user in an active pairing session emits `pairingLocationUpdate`
- **THEN** `UserPairingLocationStatus` is upserted with the latest coordinates

---

## MongoDB Schemas

### `driverlocationstatuses` collection

| Field | Type | Required | Indexed | Description |
|-------|------|----------|---------|-------------|
| `_id` | ObjectId | yes | PK | |
| `driverId` | string | yes | yes | Driver user ID |
| `location` | GeoJSON Point | no | 2dsphere | `{ type: "Point", coordinates: [lng, lat] }` |
| `accuracy` | number | no | no | GPS accuracy in metres |
| `status` | string | no | yes | `active` / `inactive` / `idle` / `assigned` / `pending` |
| `loginTime` | date | no | no | When driver went active |
| `logoutTime` | date | no | no | When driver went inactive |
| `isMoWo` | boolean | no | no | |
| `organization` | string | no | no | |
| `createdAt` | date | yes | yes | Used by stale cleanup cron |
| `updatedAt` | date | yes | no | |

**Indexes:** `driverId` (unique), `2dsphere` on `location`, `createdAt` ascending (for cron cleanup)

---

### `driverlocationtracks` collection

| Field | Type | Required | Indexed | Description |
|-------|------|----------|---------|-------------|
| `_id` | ObjectId | yes | PK | |
| `driverId` | string | yes | yes | |
| `rideId` | string | no | yes | Associated ride request ID; null when not on a trip |
| `location` | GeoJSON Point | no | 2dsphere | |
| `accuracy` | number | no | no | |
| `timestamp` | date | no | no | Client-reported timestamp |
| `createdAt` | date | yes | no | |

**Indexes:** `driverId`, `rideId`, `2dsphere` on `location`

---

### `tempdrivertrackings` collection

TTL collection — documents auto-expire 600 seconds after creation.

| Field | Type | Required | Indexed | Description |
|-------|------|----------|---------|-------------|
| `_id` | ObjectId | yes | PK | |
| `driverId` | string | yes | no | |
| `location` | GeoJSON Point | no | 2dsphere | |
| `accuracy` | number | no | no | |
| `createdAt` | date | yes | TTL 600s | Documents expire 10 minutes after creation |

**Indexes:** `2dsphere` on `location`; TTL on `createdAt` (600 s)

---

### `userpairinglocationstatuses` collection

| Field | Type | Required | Indexed | Description |
|-------|------|----------|---------|-------------|
| `_id` | ObjectId | yes | PK | |
| `userId` | string | yes | yes | |
| `pairingRequestId` | string | no | yes | |
| `location` | GeoJSON Point | no | 2dsphere | |
| `accuracy` | number | no | no | |
| `status` | string | no | no | Pairing session state |
| `createdAt` | date | yes | no | |
| `updatedAt` | date | yes | no | |

**Indexes:** `userId`, `pairingRequestId`, `2dsphere` on `location`

---

## Redis Key Reference

| Key | Type | Operations | Purpose |
|-----|------|------------|---------|
| `driver:geo` | Redis Sorted Set (Geo) | `GEOADD`, `ZREM`, `GEORADIUS`, `GEODIST` | Live driver positions for assignment geo-queries and station proximity checks |

**No other Redis keys are used by this service or any other TUTEM service.**
