## ADDED Requirements

---

# PRD: Ride Service

**Service:** `ride-service`
**Port:** 5002
**Stack:** Java 17 / Spring Boot 3
**Database:** MongoDB (dedicated `ride-service` database)
**Redis:** None
**Kafka:** Produces to `ride.events`, `notification.dispatch`, `analytics.events`; Consumes from `location.updates`
**Owned collections:** `riderequests`, `userpairingriderequests`, `drivervehicledetails`, `stations`, `fifostationdata`, `fares`, `driversearchlogs`

---

## Module: Ride Lifecycle

Manages the full state machine of a standard ride request — from booking through assignment, acceptance, in-progress, and completion or cancellation.

**Ride request status state machine:**
```
pending → assigned → driverreached → ontrip → endtrip / reacheddestination
       ↘ cancelledbyuser / cancelledbydriver / drivernotresponding
```

---

### Feature: Book a Ride

A commuter submits a ride request. The request is created in `pending` state.

**API Contract:**

| | |
|--|--|
| Method + Path | `POST /api/ride-request/book-ride` |
| Auth | JWT required (commuter) |
| Request body | `{ "originName": string, "originLat": number, "originLong": number, "destName": string, "destLat": number, "destLong": number, "rideType": string, "organization"?: string }` |
| 201 | `{ "rideRequestId": string }` |
| 400 | `{ "error": "validation_error", "details": [...] }` |
| 409 | `{ "error": "active_ride_exists" }` |

**Kafka Event Produced — `ride.events`:**
```json
{
  "eventType": "ride_created",
  "rideRequestId": "64a1f3e8c2a4b20017e4f502",
  "userId": "64a1f3e8c2a4b20017e4f503",
  "originName": "Gate 1, BITS Hyderabad",
  "originLat": 17.5449,
  "originLong": 78.5719,
  "destName": "Ameerpet Metro",
  "destLat": 17.4375,
  "destLong": 78.4483,
  "fareAmount": 120,
  "distance": 12400,
  "organization": "Personal",
  "occurredAt": "2025-03-10T14:23:00.000Z"
}
```
Topic key: `rideRequestId`

#### Scenario: Successful booking
- **WHEN** an authenticated commuter submits valid origin, destination, and ride type
- **THEN** a `RideRequest` document is created with `status: "pending"` and HTTP 201 is returned with `rideRequestId`; a `ride_created` Kafka event is published

#### Scenario: Commuter already has active ride
- **WHEN** the commuter already has a request in `pending` or `assigned` state
- **THEN** HTTP 409 is returned with `{ "error": "active_ride_exists" }`; no new document is created

#### Scenario: Invalid coordinates
- **WHEN** origin or destination coordinates are outside valid lat/lng bounds or missing
- **THEN** HTTP 400 is returned with field-level validation details

---

### Feature: Driver Assignment (Cron)

A scheduled cron (`processRideRequests`) queries the Redis geo-index via Location Service to find the nearest available driver and assigns them to a pending ride request.

**No direct HTTP endpoint — triggered by cron only.**

**Kafka Event Produced — `ride.events` (on successful assignment):**
```json
{
  "eventType": "ride_assigned",
  "rideRequestId": "64a1f3e8c2a4b20017e4f502",
  "userId": "64a1f3e8c2a4b20017e4f503",
  "driverId": "64a1f3e8c2a4b20017e4f501",
  "driverName": "Ramesh Kumar",
  "driverPhone": "9876543210",
  "vehicleMake": "Maruti",
  "vehicleModel": "Swift",
  "vehicleColor": "White",
  "vehicleRegistrationNo": "TS09AB1234",
  "uniqueCode": "4821",
  "fareAmount": 120,
  "expectedReachTime": "2025-03-10T14:38:00.000Z",
  "driverFcmToken": "fcm_token_string",
  "userFcmToken": "fcm_token_string",
  "occurredAt": "2025-03-10T14:23:00.000Z"
}
```

**Kafka Event Produced — `ride.events` (when no driver found):**
```json
{
  "eventType": "no_driver_available",
  "rideRequestId": "64a1f3e8c2a4b20017e4f502",
  "userId": "64a1f3e8c2a4b20017e4f503",
  "originLat": 17.5449,
  "originLong": 78.5719,
  "maxRadiusSearched": 5000,
  "userFcmToken": "fcm_token_string",
  "occurredAt": "2025-03-10T14:23:45.000Z"
}
```

**Cross-service dependency:** Ride Service reads driver geo-position data consumed from `location.updates` Kafka topic (produced by Location Service) and maintained in an internal MongoDB `2dsphere` index on `driverLocationStatuses` (Phase 1 read copy).

#### Scenario: Driver assigned successfully
- **WHEN** the cron finds the nearest available driver within the search radius
- **THEN** the `RideRequest` status transitions to `assigned`; a `ride_assigned` event is published; Notification Service dispatches FCM push to both parties

#### Scenario: No driver available
- **WHEN** no available driver is within the maximum search radius after all retries
- **THEN** the request remains `pending`; a `no_driver_available` event is published; the commuter receives a push notification

#### Scenario: Selected driver goes offline before assignment completes
- **WHEN** the chosen driver's location status becomes offline between geo-query and assignment write
- **THEN** the assignment for that driver is skipped; the next nearest driver is selected in the same cron cycle

---

### Feature: Driver Accept / Decline Ride

A driver explicitly accepts or declines an assigned ride request.

**API Contracts:**

**Accept:**
| | |
|--|--|
| Method + Path | `PUT /api/ride-request/accept-ride` |
| Auth | JWT required (driver) |
| Request body | `{ "rideRequestId": string }` |
| 200 | `{ "message": "ride accepted" }` |
| 403 | `{ "error": "not_assigned_to_driver" }` |
| 409 | `{ "error": "ride_already_cancelled" }` |

**Decline:**
| | |
|--|--|
| Method + Path | `PUT /api/ride-request/decline-ride` |
| Auth | JWT required (driver) |
| Request body | `{ "rideRequestId": string, "reason"?: string }` |
| 200 | `{ "message": "ride declined" }` |
| 403 | `{ "error": "not_assigned_to_driver" }` |

**Kafka Event Produced — `ride.events` (on accept):**
```json
{
  "eventType": "ride_accepted",
  "rideRequestId": "64a1f3e8c2a4b20017e4f502",
  "userId": "64a1f3e8c2a4b20017e4f503",
  "driverId": "64a1f3e8c2a4b20017e4f501",
  "occurredAt": "2025-03-10T14:24:00.000Z"
}
```

#### Scenario: Driver accepts within timeout
- **WHEN** the assigned driver calls accept-ride within the acceptance window
- **THEN** `status` transitions to `assigned` (confirmed); `ride_accepted` event is published; commuter is notified via FCM

#### Scenario: Driver declines
- **WHEN** a driver calls decline-ride
- **THEN** the request transitions back to `pending` for reassignment; the driver's ID is added to `canceledDrivers` to prevent re-assignment; no Kafka event required

#### Scenario: Request belongs to different driver
- **WHEN** a driver calls accept or decline for a rideRequestId not assigned to them
- **THEN** HTTP 403 is returned with `{ "error": "not_assigned_to_driver" }`

---

### Feature: Ride Status Transitions (Driver Reached, On Trip, End Trip)

Endpoints for the driver to advance the ride through its lifecycle.

**API Contracts:**

**Driver Reached:**
| | |
|--|--|
| Method + Path | `PUT /api/ride-request/driver-reached` |
| Auth | JWT required (driver) |
| Request body | `{ "rideRequestId": string }` |
| 200 | `{ "message": "status updated" }` |

**Start Trip:**
| | |
|--|--|
| Method + Path | `PUT /api/ride-request/start-trip` |
| Auth | JWT required (driver) |
| Request body | `{ "rideRequestId": string }` |
| 200 | `{ "message": "trip started" }` |

**Complete Trip:**
| | |
|--|--|
| Method + Path | `PUT /api/ride-request/complete-ride` |
| Auth | JWT required (driver) |
| Request body | `{ "rideRequestId": string, "finalDistance"?: number }` |
| 200 | `{ "message": "ride completed", "fareAmountCalculated": number }` |
| 409 | `{ "error": "ride_already_completed" }` |

**Kafka Event Produced — `ride.events` (on completion):**
```json
{
  "eventType": "ride_completed",
  "rideRequestId": "64a1f3e8c2a4b20017e4f502",
  "userId": "64a1f3e8c2a4b20017e4f503",
  "driverId": "64a1f3e8c2a4b20017e4f501",
  "fareAmount": 120,
  "fareAmountCalculated": 135,
  "FinalTraveldistance": 12.8,
  "startedAt": "2025-03-10T14:28:00.000Z",
  "completedAt": "2025-03-10T14:55:00.000Z",
  "driverFcmToken": "fcm_token_string",
  "userFcmToken": "fcm_token_string",
  "occurredAt": "2025-03-10T14:55:00.000Z"
}
```

#### Scenario: Trip completed successfully
- **WHEN** driver calls complete-ride on an `ontrip` request
- **THEN** status transitions to `endtrip`; fare is calculated and persisted; `ride_completed` event is published; both parties receive FCM notification

#### Scenario: Complete called on already-completed ride
- **WHEN** complete-ride is called for a request already in `endtrip` state
- **THEN** HTTP 409 is returned; no duplicate fare record is created

---

### Feature: Cancel Ride

Either the commuter or driver can cancel a ride before it is completed.

**API Contract:**
| | |
|--|--|
| Method + Path | `PUT /api/ride-request/cancel-ride` |
| Auth | JWT required (commuter or driver) |
| Request body | `{ "rideRequestId": string, "reason"?: string }` |
| 200 | `{ "message": "ride cancelled" }` |
| 409 | `{ "error": "ride_already_cancelled" }` or `{ "error": "ride_already_completed" }` |

**Kafka Event Produced — `ride.events`:**
```json
{
  "eventType": "ride_cancelled",
  "rideRequestId": "64a1f3e8c2a4b20017e4f502",
  "userId": "64a1f3e8c2a4b20017e4f503",
  "driverId": "64a1f3e8c2a4b20017e4f501",
  "cancelledBy": "user",
  "reasonForCancelling": "Driver taking too long",
  "cancelledAt": "2025-03-10T14:30:00.000Z",
  "driverFcmToken": "fcm_token_string",
  "userFcmToken": "fcm_token_string",
  "occurredAt": "2025-03-10T14:30:00.000Z"
}
```

#### Scenario: Commuter cancels before driver arrives
- **WHEN** a commuter cancels an `assigned` ride before `driverreached`
- **THEN** status transitions to `cancelledbyuser`; `ride_cancelled` event is published with `cancelledBy: "user"`

#### Scenario: Concurrent cancellations
- **WHEN** both commuter and driver call cancel simultaneously
- **THEN** first write wins via `findOneAndUpdate` status filter; second call receives HTTP 409

---

### Feature: Stale Ride Cleanup (Cron)

A cron job transitions `pending` requests that exceed their timeout to `cancelled`.

**No direct HTTP endpoint — triggered by cron only.**

**Cron:** `rideMonitor` — runs on a configurable interval.

#### Scenario: Stale pending request cleaned up
- **WHEN** a `RideRequest` has been in `pending` state longer than the configured timeout
- **THEN** status transitions to `cancelled` with reason `timeout`; a `ride_cancelled` event is published; the commuter is notified

#### Scenario: Race condition with driver accept
- **WHEN** the cron targets a request that is being accepted by a driver simultaneously
- **THEN** the `findOneAndUpdate` with `status: "pending"` filter ensures only one write wins; if accept wins, cleanup skips that document

---

## Module: Driver Management

Manages driver vehicle details, availability status, and driver search log.

---

### Feature: Register / Update Vehicle Details

**API Contracts:**

**Register Vehicle:**
| | |
|--|--|
| Method + Path | `POST /api/driver-vehicle/register` |
| Auth | JWT required (driver) |
| Request body | `{ "vehicleMake": string, "vehicleModel": string, "vehicleColor": string, "registrationNo": string, "vehicleType": string }` |
| 201 | `{ "vehicleId": string }` |
| 409 | `{ "error": "vehicle_already_registered" }` |

**Update Vehicle:**
| | |
|--|--|
| Method + Path | `PUT /api/driver-vehicle/:vehicleId` |
| Auth | JWT required (driver, owns vehicle) |
| Request body | `{ "vehicleColor"?: string, "vehicleModel"?: string }` |
| 200 | `{ "message": "vehicle updated" }` |

#### Scenario: Driver registers vehicle
- **WHEN** a driver submits valid vehicle details
- **THEN** a `DriverVehicleDetail` document is created and `vehicleId` is returned

---

### Feature: Driver Search Log

Records each driver search attempt (nearest-driver geo-query) for admin visibility and debugging.

**No direct HTTP endpoint — written by the assignment cron.**

**Kafka Event Produced — `analytics.events`:**
```json
{
  "eventType": "driver_search_performed",
  "rideRequestId": "64a1f3e8c2a4b20017e4f502",
  "searchRadius": 3000,
  "driversFound": 2,
  "driverAssigned": "64a1f3e8c2a4b20017e4f501",
  "occurredAt": "2025-03-10T14:23:00.000Z"
}
```

#### Scenario: Search log written on every assignment attempt
- **WHEN** the assignment cron runs for a pending request
- **THEN** a `DriverSearchLog` document is written regardless of whether a driver was found; an `analytics.events` event is published

---

## Module: Fare Management

Manages fare configuration and fare calculation for completed rides.

---

### Feature: Fare Configuration (Admin)

**API Contracts:**

**Create Fare Rule:**
| | |
|--|--|
| Method + Path | `POST /api/fareamount` |
| Auth | JWT required (admin) |
| Request body | `{ "vehicleType": string, "baseAmount": number, "perKmRate": number, "minimumFare": number }` |
| 201 | `{ "fareId": string }` |

**Get All Fare Rules:**
| | |
|--|--|
| Method + Path | `GET /api/fareamount` |
| Auth | JWT required (admin) |
| 200 | Array of fare rule objects |

#### Scenario: Admin creates fare rule
- **WHEN** an admin submits a fare rule for a vehicle type that does not yet exist
- **THEN** the rule is created and `fareId` is returned

#### Scenario: Missing fare config at trip completion
- **WHEN** no fare rule exists for a vehicle type when a ride completes
- **THEN** the default fallback rate is applied; the anomaly is flagged on the ride record; the ride still completes successfully

---

### Feature: Fare Calculation

Fare is computed when a ride transitions to `endtrip`.

**No direct HTTP endpoint — triggered internally on ride completion.**

Calculation formula: `max(minimumFare, baseAmount + (distance_km × perKmRate))`

#### Scenario: Fare calculated on completion
- **WHEN** a ride transitions to `endtrip`
- **THEN** fare is computed using the vehicle type's `FareAmount` rule and the actual covered distance; the result is persisted to `fareAmountCalculated` on the `RideRequest`

---

## Module: Station Queue (FIFO)

Manages station-based ride queues for pre-defined pickup/drop-off stations.

---

### Feature: Station Check-In

**API Contracts:**

**Commuter Check-In:**
| | |
|--|--|
| Method + Path | `POST /api/stations/checkin` |
| Auth | JWT required (commuter) |
| Request body | `{ "stationId": string }` |
| 200 | `{ "queuePosition": number }` |
| 404 | `{ "error": "station_not_found" }` |

**Driver Arrival at Station:**
| | |
|--|--|
| Method + Path | `POST /api/stations/driver-arrived` |
| Auth | JWT required (driver) |
| Request body | `{ "stationId": string }` |
| 200 | `{ "message": "matched" }` or `{ "message": "added to waiting pool" }` |

#### Scenario: Driver arrives, commuter in queue
- **WHEN** a driver arrives at a station that has at least one commuter in queue
- **THEN** the first commuter in FIFO order is matched to the driver; `FifoStationData` is updated; both are notified via Kafka event on `notification.dispatch`

#### Scenario: Driver arrives, empty queue
- **WHEN** a driver arrives at a station with no commuters waiting
- **THEN** the driver is added to the driver-waiting pool; the next commuter to check in is matched immediately

#### Scenario: Commuter leaves queue
- **WHEN** a commuter cancels their station check-in
- **THEN** their entry is removed from `FifoStationData`; queue positions are compacted; no phantom match occurs

---

## Module: User-to-User Pairing (Carpooling)

Manages carpooling requests and OSMnx-based route matching.

---

### Feature: Create and Match Pairing Request

**API Contracts:**

**Create Pairing Request:**
| | |
|--|--|
| Method + Path | `POST /api/pairing-ride/request` |
| Auth | JWT required (commuter) |
| Request body | `{ "originLat": number, "originLong": number, "destLat": number, "destLong": number, "departureWindowStart": string, "departureWindowEnd": string }` |
| 201 | `{ "pairingRequestId": string }` |

**Cancel Pairing Request:**
| | |
|--|--|
| Method + Path | `DELETE /api/pairing-ride/:pairingRequestId` |
| Auth | JWT required (commuter, owns request) |
| 204 | *(empty)* |

**Cross-service dependency:** Route matching invokes a Python subprocess (`match_rider.py` via OSMnx). This is a Ride Service internal call — not a service-to-service HTTP call.

#### Scenario: Matching pairing request found
- **WHEN** a compatible open pairing request exists (overlapping route, within departure window)
- **THEN** `match_rider.py` is invoked; compatible requests are paired; both commuters are notified via `notification.dispatch` Kafka event

#### Scenario: Python subprocess fails
- **WHEN** `match_rider.py` times out or throws an exception
- **THEN** the pairing attempt is logged as failed; the request remains `pending` for retry on the next cron cycle; the Node process does not crash

#### Scenario: Commuter cancels before match
- **WHEN** a commuter cancels their pairing request before it is matched
- **THEN** the request is removed from the matching pool; HTTP 204 is returned

---

## MongoDB Schemas

### `riderequests` collection

| Field | Type | Required | Indexed | Description |
|-------|------|----------|---------|-------------|
| `_id` | ObjectId | yes | PK | |
| `userId` | string | yes | yes | Commuter user ID |
| `driverId` | string | no | yes | Assigned driver user ID |
| `status` | string | yes | yes | Ride state machine value |
| `originName` | string | yes | no | Human-readable origin |
| `originLat` | number | yes | no | |
| `originLong` | number | yes | no | |
| `destName` | string | yes | no | |
| `destLat` | number | yes | no | |
| `destLong` | number | yes | no | |
| `fareAmount` | number | no | no | Pre-calculated fare at booking |
| `fareAmountCalculated` | number | no | no | Final fare after trip |
| `distance` | number | no | no | Pre-calculated distance (metres) |
| `FinalTraveldistance` | number | no | no | Actual covered distance (km) |
| `uniqueCode` | string | no | no | 4-digit shared verification code |
| `canceledDrivers` | string[] | no | no | Driver IDs that declined |
| `reasonForCancelling` | string | no | no | |
| `isMoWo` | boolean | no | no | MoWo fleet ride flag |
| `organization` | string | no | no | |
| `startedAt` | date | no | no | |
| `completedAt` | date | no | no | |
| `cancelledAt` | date | no | no | |
| `driverReachedAt` | date | no | no | |
| `onTripAt` | date | no | no | |
| `expectedReachTime` | date | no | no | ETA at pickup |
| `createdAt` | date | yes | yes | Auto-managed |
| `updatedAt` | date | yes | no | Auto-managed |

**Indexes:** `userId`, `driverId`, `status`, `createdAt`

### `drivervehicledetails` collection

| Field | Type | Required | Indexed | Description |
|-------|------|----------|---------|-------------|
| `_id` | ObjectId | yes | PK | |
| `driverId` | string | yes | yes (unique) | One vehicle per driver |
| `vehicleMake` | string | yes | no | |
| `vehicleModel` | string | yes | no | |
| `vehicleColor` | string | yes | no | |
| `registrationNo` | string | yes | yes (unique) | |
| `vehicleType` | string | yes | yes | Used for fare lookup |

### `fares` collection

| Field | Type | Required | Indexed | Description |
|-------|------|----------|---------|-------------|
| `_id` | ObjectId | yes | PK | |
| `vehicleType` | string | yes | yes (unique) | Fare rule key |
| `baseAmount` | number | yes | no | |
| `perKmRate` | number | yes | no | |
| `minimumFare` | number | yes | no | |
