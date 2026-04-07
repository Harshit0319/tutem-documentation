## ADDED Requirements

---

# PRD: Notification Service

**Service:** `notification-service`
**Port:** 5005
**Stack:** Java 17 / Spring Boot 3
**Database:** None in Phase 1 (stateless — no owned collections). A `notificationlogs` collection is a Phase 2 consideration.
**Redis:** None
**Kafka:** Consumes from `notification.dispatch`, `ride.events`, `user.events`
**External dependencies:** Firebase Cloud Messaging (FCM), SMTP provider
**Owned collections:** None (Phase 1)

---

## Module: Push Notification Dispatch

Consumes `notification.dispatch` Kafka events and delivers FCM push notifications to target device tokens.

---

### Feature: FCM Push Notification

**Kafka Event Consumed — `notification.dispatch` → `push_notification`:**
```json
{
  "eventType": "push_notification",
  "targetUserId": "64a1f3e8c2a4b20017e4f503",
  "fcmToken": "fcm_token_string_here",
  "title": "Driver is on the way",
  "body": "Your driver Ramesh is 5 minutes away. Code: 4821",
  "data": {
    "rideId": "64a1f3e8c2a4b20017e4f502",
    "screen": "ride_tracking"
  },
  "occurredAt": "2025-03-10T14:24:00.000Z"
}
```

**On receipt, the service SHALL:**
1. Extract `fcmToken`, `title`, `body`, and optional `data` payload
2. Call the FCM send API
3. Log the outcome: `{ event: "fcm_sent", targetUserId, messageId, occurredAt }` on success
4. Log the error: `{ event: "fcm_failed", targetUserId, errorCode, occurredAt }` on failure
5. Commit the Kafka offset after the attempt (success or handled failure)

**Field constraints:**
- `title` max 65 characters
- `body` max 240 characters
- `data` payload must not exceed FCM's 4 KB limit; if exceeded, strip `data` and send title + body only

#### Scenario: FCM push delivered successfully
- **WHEN** a `push_notification` event is consumed with a valid `fcmToken`, `title`, and `body`
- **THEN** the notification is dispatched to FCM; success is logged with `messageId`; Kafka offset is committed

#### Scenario: Invalid or unregistered FCM token
- **WHEN** FCM returns `registration-token-not-registered` or `invalid-registration-token`
- **THEN** the error is logged with the token's last 4 characters; the Kafka offset is committed (no retry); a `device_token_invalid` log entry is written for User Service to act on in a future cleanup job

#### Scenario: FCM temporarily unavailable
- **WHEN** FCM returns a 5xx or network timeout
- **THEN** the Kafka offset is NOT committed; the event is retried on the next poll cycle up to the configured max retries; after max retries, the event is sent to a dead-letter topic and the offset is committed

#### Scenario: Data payload exceeds 4 KB
- **WHEN** the `data` field causes the total FCM payload to exceed 4 KB
- **THEN** the `data` field is stripped; the notification is sent with `title` and `body` only; a warning is logged

---

### Feature: Notification Dispatch for Ride Events

Listens to `ride.events` and dispatches FCM pushes and emails for each ride lifecycle transition.

**Kafka Events Consumed — `ride.events`:**

| `eventType` | FCM Push | Email |
|-------------|----------|-------|
| `ride_assigned` | Yes — to both driver and commuter | No |
| `ride_accepted` | Yes — to commuter | No |
| `ride_completed` | Yes — to both | Yes — to commuter (if email registered) |
| `ride_cancelled` | Yes — to both | No |
| `no_driver_available` | Yes — to commuter | No |

**FCM dispatch for `ride_assigned` example:**

Commuter push:
```json
{
  "title": "Driver assigned",
  "body": "Ramesh is on the way. Code: 4821. ETA: 15 min",
  "data": { "rideId": "<id>", "screen": "ride_tracking" }
}
```

Driver push:
```json
{
  "title": "New ride request",
  "body": "Pick up at Gate 1, BITS Hyderabad. Code: 4821",
  "data": { "rideId": "<id>", "screen": "ride_detail" }
}
```

#### Scenario: Ride assigned — both parties notified
- **WHEN** a `ride_assigned` event is consumed
- **THEN** an FCM push is sent to the commuter's FCM token and the driver's FCM token; if either token is null, that push is skipped and logged

#### Scenario: Ride completed — email sent
- **WHEN** a `ride_completed` event is consumed and the commuter has a registered email
- **THEN** a ride-complete email is sent via SMTP with fare and trip summary; if the commuter has no email, the email step is skipped and the event is still acknowledged on Kafka

#### Scenario: FCM token missing
- **WHEN** a ride event is consumed but `userFcmToken` or `driverFcmToken` is null
- **THEN** the push for the party with the null token is skipped; the other party's push is still sent; no error is thrown

---

## Module: Email Dispatch

Sends transactional emails for ride lifecycle events and account-related actions via SMTP.

---

### Feature: Transactional Email (Ride Events)

**Kafka Event Consumed — `notification.dispatch` → `email_notification`:**
```json
{
  "eventType": "email_notification",
  "targetUserId": "64a1f3e8c2a4b20017e4f503",
  "to": "priya@example.com",
  "subject": "TUTEM: Your ride is complete",
  "htmlBody": "<h1>Your ride is complete</h1><p>Fare: ₹135</p>",
  "templateId": "ride_complete",
  "occurredAt": "2025-03-10T14:55:00.000Z"
}
```

If `htmlBody` is present, it is used directly. If absent, `templateId` is used to render a server-side template.

#### Scenario: Email sent successfully
- **WHEN** an `email_notification` event is consumed with a valid `to` address
- **THEN** the email is sent via SMTP; success is logged; Kafka offset is committed

#### Scenario: SMTP server unreachable
- **WHEN** the SMTP connection fails
- **THEN** the Kafka offset is NOT committed; the event is retried; after max retries, the event is sent to a dead-letter topic

#### Scenario: Template not found
- **WHEN** `templateId` is specified but no matching template exists
- **THEN** a plain-text fallback email is sent using `subject` and a generic body; a warning is logged; the Kafka offset is committed

---

### Feature: Password Reset Email

Sends a time-limited password reset link to the user's registered email address.

**API Contract:**

**Request Reset:**
| | |
|--|--|
| Method + Path | `POST /api/auth/forgot-password` |
| Auth | Public |
| Request body | `{ "email": string }` |
| 200 | `{ "message": "if registered, a reset email has been sent" }` |

Note: HTTP 200 is always returned regardless of whether the email is registered, to prevent email enumeration.

**Reset Password:**
| | |
|--|--|
| Method + Path | `POST /api/auth/reset-password` |
| Auth | Public (reset token in body) |
| Request body | `{ "token": string, "newPassword": string }` |
| 200 | `{ "message": "password reset successful" }` |
| 401 | `{ "error": "reset_token_expired" }` or `{ "error": "reset_token_already_used" }` |
| 400 | `{ "error": "validation_error", "details": [...] }` |

Reset token: signed JWT, TTL 15 minutes. Invalidated after first use.

**Cross-service dependency:** This endpoint is proxied through API Gateway. The reset token is issued by User Service (which owns the `users` collection). Notification Service consumes a `user.events` → `password_reset_requested` event and sends the email.

**Kafka Event Consumed — `user.events` → `password_reset_requested`:**
```json
{
  "eventType": "password_reset_requested",
  "userId": "64a1f3e8c2a4b20017e4f503",
  "email": "priya@example.com",
  "resetToken": "<signed JWT>",
  "expiresAt": "2025-03-10T14:38:00.000Z",
  "occurredAt": "2025-03-10T14:23:00.000Z"
}
```

#### Scenario: Registered email — reset link sent
- **WHEN** `POST /api/auth/forgot-password` is called with a registered email
- **THEN** User Service publishes a `password_reset_requested` event; Notification Service consumes it and sends the reset email; HTTP 200 is returned

#### Scenario: Unregistered email — no enumeration
- **WHEN** `POST /api/auth/forgot-password` is called with an email that is not registered
- **THEN** HTTP 200 is returned; no email is sent; no event is published; a warning is logged internally

#### Scenario: Expired reset token
- **WHEN** `POST /api/auth/reset-password` is called with a token past its 15-minute TTL
- **THEN** HTTP 401 is returned with `{ "error": "reset_token_expired" }`

#### Scenario: Reset token reuse attempt
- **WHEN** `POST /api/auth/reset-password` is called with a token that has already been used
- **THEN** HTTP 401 is returned with `{ "error": "reset_token_already_used" }`

---

## Module: Service Health

---

### Feature: Health Check

**API Contract:**

| | |
|--|--|
| Method + Path | `GET /health` |
| Auth | Public |
| 200 | `{ "status": "ok", "service": "notification-service", "fcm": "initialised", "smtp": "ready" }` |
| 503 | `{ "status": "degraded", "service": "notification-service", "fcm": "error", "smtp": "ready" }` |

The health check reflects the initialisation state of both FCM and SMTP. If either fails to initialise, the response is `503` and Kubernetes readiness probe marks the pod as not ready.

#### Scenario: All dependencies initialised
- **WHEN** GET /health is called and both FCM and SMTP have initialised successfully
- **THEN** HTTP 200 is returned with `status: "ok"` and both dependency states as `"initialised"` / `"ready"`

#### Scenario: FCM fails to initialise
- **WHEN** the FCM service account JSON is invalid or missing
- **THEN** GET /health returns HTTP 503 with `{ "status": "degraded", "fcm": "error" }`; the service continues processing email-only events but skips FCM dispatch

---

## Kafka Consumer Summary

| Topic | Event types consumed | Action |
|-------|---------------------|--------|
| `notification.dispatch` | `push_notification` | Send FCM push |
| `notification.dispatch` | `email_notification` | Send SMTP email |
| `ride.events` | `ride_assigned`, `ride_accepted`, `ride_completed`, `ride_cancelled`, `no_driver_available` | Dispatch FCM and/or email per event type |
| `user.events` | `password_reset_requested`, `device_token_updated` | Send reset email; update internal token reference (Phase 2) |
