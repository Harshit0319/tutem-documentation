## Why

The TUTEM platform has a system design layer and an extraction plan but no shared standards document that engineers can reference when building any of the 6 microservices. Without a canonical API protocol, folder structure, per-service PRDs, and a dependency map, each engineer makes independent decisions that diverge silently — leading to inconsistent error shapes, undocumented inter-service contracts, and onboarding friction for interns.

## What Changes

- **NEW** `api-protocol.md` — Platform-wide API standard (JSON:API-inspired, HTTP semantics per RFC 7231, error contract, versioning, pagination)
- **NEW** `folder-structure-standard.md` — Hexagonal architecture layout mirroring `user-service/FOLDER_STRUCTURE.md`, applicable to all services
- **NEW** Per-service PRD files (one per service): `prd-api-gateway.md`, `prd-user-service.md`, `prd-ride-service.md`, `prd-location-service.md`, `prd-notification-service.md`, `prd-analytics-service.md`
- **NEW** `prd-index.md` — Single-entry knowledge graph / context map so AI tooling and engineers know exactly which PRD file owns each module and feature
- **NEW** `cross-service-dependency-map.md` — Explicit mapping of features that span more than one service, including which API calls and Kafka events are required
- **NEW** `roadmap.md` — Phased delivery roadmap sequenced for a team of 2-3 senior engineers + 2 interns
- Redis references are scoped strictly: Redis is only included for geo-indexing in Location Service; no Redis caching in any other service

## Capabilities

### New Capabilities

- `api-protocol`: Platform-wide HTTP API standard (RFC 7231 semantics, JSON error contract, versioning, pagination, status code table)
- `folder-structure-standard`: Canonical hexagonal folder structure for all services, derived from user-service reference
- `prd-api-gateway`: PRD for API Gateway — routing, CORS, health, proxy behaviour, per-module and per-feature breakdown with API contracts
- `prd-user-service`: PRD for User Service — auth, OTP, profile, admin, per-module and per-feature breakdown with API and Kafka contracts
- `prd-ride-service`: PRD for Ride Service — ride lifecycle, driver management, pairing, fare, stations, per-module and per-feature breakdown with API and Kafka contracts
- `prd-location-service`: PRD for Location Service — real-time tracking, geo-indexing (Redis), Socket.io, per-module and per-feature breakdown with API and Kafka contracts
- `prd-notification-service`: PRD for Notification Service — FCM push, email dispatch, per-module and per-feature breakdown with API and Kafka contracts
- `prd-analytics-service`: PRD for Analytics Service — dashboards, summaries, cron jobs, per-module and per-feature breakdown with API and Kafka contracts
- `prd-index`: Single-entry knowledge graph — maps every module/feature to its PRD file and section anchor
- `cross-service-dependency-map`: Mapping of all features with multi-service dependencies, including event and API contracts at each boundary
- `roadmap`: Phased delivery plan for 2-3 senior engineers + 2 interns, sequenced by dependency order and parallelisability

### Modified Capabilities

## Impact

- All 6 services: API Gateway, User Service, Ride Service, Location Service, Notification Service, Analytics Service
- Existing `system-design-layer` and `prd-and-epic-mapping` changes are referenced as source-of-truth; this change produces the actionable engineering standards layer on top of them
- No breaking changes to existing service contracts — this is a documentation and standards change only
- Redis dependency is REMOVED from all services except Location Service (geo-index only)
