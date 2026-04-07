## ADDED Requirements

---

# PRD: User Service

**Service:** `user-service`
**Port:** 5001
**Stack:** Java 17 / Spring Boot 3
**Database:** MongoDB (dedicated `user-service` database)
**Redis:** None
**Kafka:** Produces to `user.events`, `notification.dispatch`
**Owned collections:** `users`

---

## Module: Authentication

Handles user registration, OTP-based login, MPIN-based login, JWT issuance, and logout.

---

### Feature: User Registration

A new user account is created with phone number, name, and role.

**API Contract:**

| | |
|--|--|
| Method + Path | `POST /api/auth/register` |
| Auth | Public |
| Request body | `{ "phone": string, "name": string, "role": "commuter" \| "driver" }` |
| 201 | `{ "userId": string }` |
| 400 | `{ "error": "validation_error", "details": [...] }` — missing/invalid fields |
| 409 | `{ "error": "phone_already_registered" }` |

**Kafka Event Produced — `user.events`:**
```json
{
  "eventType": "user_registered",
  "userId": "<string>",
  "phone": "<string>",
  "role": "commuter | driver",
  "occurredAt": "<ISO8601>"
}
```
Topic key: `userId`

#### Scenario: Successful registration
- **WHEN** POST /api/auth/register is called with valid phone, name, and role
- **THEN** a User document is created in MongoDB; HTTP 201 is returned with the new userId; a `user_registered` event is published to `user.events`

#### Scenario: Duplicate phone number
- **WHEN** a phone number that is already registered is submitted
- **THEN** HTTP 409 is returned with `{ "error": "phone_already_registered" }`; no duplicate document is created; no Kafka event is emitted

#### Scenario: Invalid role value
- **WHEN** role is not `commuter` or `driver`
- **THEN** HTTP 400 is returned with `{ "error": "validation_error", "details": [{ "field": "role", "issue": "must be commuter or driver" }] }`

---

### Feature: OTP Send and Verify

Sends an OTP to the user's phone and verifies it to issue a JWT.

**API Contracts:**

**Send OTP:**
| | |
|--|--|
| Method + Path | `POST /api/auth/send-otp` |
| Auth | Public |
| Request body | `{ "phone": string }` |
| 200 | `{ "message": "OTP sent" }` |
| 404 | `{ "error": "user_not_found" }` |
| 429 | `{ "error": "otp_rate_limited" }` |

**Verify OTP:**
| | |
|--|--|
| Method + Path | `POST /api/auth/verify-otp` |
| Auth | Public |
| Request body | `{ "phone": string, "otp": string }` |
| 200 | `{ "accessToken": string, "refreshToken": string, "userId": string }` |
| 401 | `{ "error": "otp_invalid" }` or `{ "error": "otp_expired" }` |
| 404 | `{ "error": "user_not_found" }` |

OTP is stored as a hashed value in the `users` document with a TTL field. Rate limit: max 5 OTP requests per phone number per 10 minutes — enforced in-memory (no Redis).

#### Scenario: Successful OTP verification
- **WHEN** a valid OTP is submitted within its validity window
- **THEN** HTTP 200 is returned with a signed JWT access token and refresh token; the OTP is invalidated (cleared from the document)

#### Scenario: Expired OTP
- **WHEN** an OTP is submitted after its validity window has elapsed
- **THEN** HTTP 401 is returned with `{ "error": "otp_expired" }`

#### Scenario: OTP rate limit exceeded
- **WHEN** more than 5 OTP requests are made for the same phone within 10 minutes
- **THEN** HTTP 429 is returned; no OTP is sent

---

### Feature: MPIN Login

Authenticates a user with a 4–6 digit MPIN as an alternative to OTP.

**API Contracts:**

**Set MPIN:**
| | |
|--|--|
| Method + Path | `POST /api/auth/set-mpin` |
| Auth | JWT required |
| Request body | `{ "mpin": string }` |
| 200 | `{ "message": "MPIN set" }` |
| 400 | `{ "error": "invalid_mpin_format" }` |

**Login with MPIN:**
| | |
|--|--|
| Method + Path | `POST /api/auth/mpin-login` |
| Auth | Public |
| Request body | `{ "phone": string, "mpin": string }` |
| 200 | `{ "accessToken": string, "refreshToken": string, "userId": string }` |
| 401 | `{ "error": "mpin_invalid" }` |
| 403 | `{ "error": "mpin_not_set" }` or `{ "error": "account_inactive" }` |

MPIN is stored as a bcrypt hash.

#### Scenario: Successful MPIN login
- **WHEN** a registered user submits correct phone and MPIN
- **THEN** HTTP 200 is returned with a signed JWT

#### Scenario: MPIN not set
- **WHEN** a user who has never set an MPIN attempts MPIN login
- **THEN** HTTP 403 is returned with `{ "error": "mpin_not_set" }`

---

### Feature: JWT Refresh

Issues a new access token given a valid refresh token.

**API Contract:**
| | |
|--|--|
| Method + Path | `POST /api/auth/refresh-token` |
| Auth | Public (refresh token in body) |
| Request body | `{ "refreshToken": string }` |
| 200 | `{ "accessToken": string }` |
| 401 | `{ "error": "refresh_token_invalid" }` or `{ "error": "refresh_token_expired" }` |

JWT config: access token TTL = 15 minutes; refresh token TTL = 7 days.
No Redis blacklist in Phase 1. Logout is client-side token discard.

#### Scenario: Valid refresh token
- **WHEN** a valid, unexpired refresh token is submitted
- **THEN** a new access token is returned; the refresh token is NOT rotated (Phase 1)

#### Scenario: Expired refresh token
- **WHEN** a refresh token past its 7-day TTL is submitted
- **THEN** HTTP 401 is returned with `{ "error": "refresh_token_expired" }`

---

## Module: User Profile

Manages reading and updating the user's profile data.

---

### Feature: Get and Update Profile

**API Contracts:**

**Get Profile:**
| | |
|--|--|
| Method + Path | `GET /api/user/profile` |
| Auth | JWT required |
| 200 | `{ "userId": string, "name": string, "phone": string, "role": string, "photoUrl": string \| null, "preferences": object }` |
| 401 | `{ "error": "token_invalid" }` |

**Update Profile:**
| | |
|--|--|
| Method + Path | `PUT /api/user/profile` |
| Auth | JWT required |
| Request body | `{ "name"?: string, "photoUrl"?: string, "preferences"?: object }` |
| 200 | `{ "message": "profile updated" }` |
| 403 | `{ "error": "role_change_not_permitted" }` — if `role` field is included in body |
| 413 | `{ "error": "file_too_large" }` — if photo URL points to an oversized file |

Role is immutable after registration. Any request body containing `role` is rejected.

#### Scenario: Successful profile fetch
- **WHEN** GET /api/user/profile is called with a valid JWT
- **THEN** HTTP 200 is returned with the user's current profile fields

#### Scenario: Role change attempt blocked
- **WHEN** PUT /api/user/profile is called with a `role` field in the body
- **THEN** HTTP 403 is returned with `{ "error": "role_change_not_permitted" }`

---

### Feature: Device Token Registration

Stores an FCM device token for push notification delivery.

**API Contract:**
| | |
|--|--|
| Method + Path | `POST /api/user/device-token` |
| Auth | JWT required |
| Request body | `{ "deviceToken": string, "platform": "ios" \| "android" }` |
| 200 | `{ "message": "device token registered" }` |
| 400 | `{ "error": "invalid_device_token" }` |

Idempotent: submitting the same token twice does not create a duplicate.

**Kafka Event Produced — `notification.dispatch`:**
```json
{
  "eventType": "device_token_registered",
  "userId": "<string>",
  "deviceToken": "<string>",
  "platform": "ios | android",
  "occurredAt": "<ISO8601>"
}
```
Topic key: `userId`

#### Scenario: Device token stored idempotently
- **WHEN** the same device token is submitted twice for the same user
- **THEN** no duplicate entry is created; HTTP 200 is returned on both calls

#### Scenario: Null token rejected
- **WHEN** an empty or null device token is submitted
- **THEN** HTTP 400 is returned with `{ "error": "invalid_device_token" }`

---

### Feature: Shuttle Opt-in

Allows commuters to opt into the shuttle service programme.

**API Contracts:**

**Opt In:**
| | |
|--|--|
| Method + Path | `POST /api/user/shuttle-optin` |
| Auth | JWT required |
| 200 | `{ "message": "opted in" }` |
| 403 | `{ "error": "not_eligible" }` — if user role is not `commuter` |

**Opt Out:**
| | |
|--|--|
| Method + Path | `POST /api/user/shuttle-optout` |
| Auth | JWT required |
| 200 | `{ "message": "opted out" }` |

Idempotent: opt-in when already opted-in returns HTTP 200 without error.

#### Scenario: Commuter opts in
- **WHEN** an authenticated commuter calls POST /api/user/shuttle-optin
- **THEN** the shuttle opt-in flag is set to true; HTTP 200 is returned

#### Scenario: Driver cannot opt in
- **WHEN** an authenticated driver calls POST /api/user/shuttle-optin
- **THEN** HTTP 403 is returned with `{ "error": "not_eligible" }`

---

## Module: Admin

Handles admin-specific authentication and user management operations.

---

### Feature: Admin Login

**API Contract:**
| | |
|--|--|
| Method + Path | `POST /api/auth/admin-login` |
| Auth | Public |
| Request body | `{ "username": string, "password": string }` |
| 200 | `{ "accessToken": string, "role": "admin" }` |
| 401 | `{ "error": "invalid_credentials" }` |

Admin credentials are stored in the `users` collection with role `admin`. Admin JWT includes a `role: admin` claim used by downstream services for authorisation.

#### Scenario: Successful admin login
- **WHEN** valid admin credentials are submitted
- **THEN** HTTP 200 is returned with a JWT that includes `role: admin` in the payload

#### Scenario: Invalid admin credentials
- **WHEN** an incorrect username or password is submitted
- **THEN** HTTP 401 is returned; no information about which field was wrong is disclosed

---

## MongoDB Schema: `users` collection

| Field | Type | Required | Indexed | Description |
|-------|------|----------|---------|-------------|
| `_id` | ObjectId | yes | yes (PK) | MongoDB primary key |
| `phone` | string | yes | yes (unique) | E.164 format |
| `name` | string | yes | no | Display name |
| `role` | string | yes | yes | `commuter`, `driver`, or `admin`; immutable after creation |
| `mpinHash` | string | no | no | bcrypt hash of MPIN |
| `otpHash` | string | no | no | bcrypt hash of current OTP |
| `otpExpiresAt` | date | no | no | OTP expiry timestamp |
| `otpRequestCount` | number | no | no | Rolling OTP request count for rate limiting |
| `otpRateLimitWindowStart` | date | no | no | Start of current OTP rate limit window |
| `deviceTokens` | array | no | no | Array of `{ token, platform, updatedAt }` |
| `photoUrl` | string | no | no | Profile photo URL |
| `preferences` | object | no | no | User preference flags |
| `shuttleOptIn` | boolean | no | no | Shuttle programme opt-in flag |
| `isActive` | boolean | yes | no | Account active flag; defaults to `true` |
| `createdAt` | date | yes | no | Account creation timestamp |
| `updatedAt` | date | yes | no | Last update timestamp |
