## Context

The TUTEM platform is being decomposed from a Node.js monolith into 6 microservices: API Gateway, User Service, Ride Service, Location Service, Notification Service, and Analytics Service. The system design layer (`system-design-layer` change) and the extraction tasks (`monolith-to-microservices` change) exist but there is no single reference that answers: "How should I structure my service folder?", "What HTTP status code do I return for a validation error?", or "Where do I find the complete PRD for the service I'm building?"

Without these standards, a team of 5 engineers (2-3 seniors, 2 interns) building in parallel will produce divergent error shapes, inconsistent folder layouts, and fragmented documentation that AI tooling cannot navigate without reading 10+ files.

This change produces the **Platform Standards and PRD layer** — a set of engineering references that sit above individual service implementations and below the system design layer.

**Constraints:**
- Redis is ONLY permitted in Location Service for geo-indexing (GEOADD/GEORADIUS). All other Redis usage (JWT blacklist, rate limiting, Socket.io adapter, Redlock) is removed from Phase 1.
- All services follow the hexagonal architecture folder layout defined in `user-service/FOLDER_STRUCTURE.md` exactly.
- Java/Spring Boot is used for User Service (reference implementation). Other services may use Node.js as indicated by the monolith extraction plan.
- Conference Service is explicitly out of scope.

---

## Goals / Non-Goals

**Goals:**
- Define a single, easy-to-follow HTTP API standard (RFC 7231 semantics) with JSON error contract, versioning, status code table, and pagination
- Define a canonical folder structure for all services derived from the user-service reference
- Produce 6 per-service PRD files with module → feature → API/event contract breakdown
- Produce a PRD index file that maps every module/feature to its file and section anchor (AI-navigable)
- Produce a cross-service dependency map showing which features require coordination across service boundaries
- Produce a delivery roadmap sequenced for 2-3 senior engineers + 2 interns
- Remove Redis from all services except Location Service geo-indexing

**Non-Goals:**
- These documents do not replace code or tests
- Infrastructure configuration (Docker, Kubernetes, Terraform) is not produced here
- Phase 2 concerns (database-per-service, service mesh, event sourcing) are not included
- Analytics Service cron job scheduling details are not in scope here (covered in monolith-to-microservices change)
- Conference Service is not included in any document

---

## Decisions

### D1 — API Standard: RFC 7231 + OpenAPI 3.0 style conventions (not full JSON:API)

**Decision:** The platform API standard is grounded in RFC 7231 (HTTP Semantics) for status codes, method semantics, and header usage. Error responses follow a minimal, consistent JSON contract: `{ "error": "<code>", "message": "<human text>", "details": [...] }`. Full JSON:API (RFC 8259 / jsonapi.org) is explicitly not adopted.

**Rationale:** JSON:API requires `data`/`relationships`/`included` envelope that adds friction for mobile clients and contradicts existing PRD response shapes. The existing PRD already defines flat response shapes (`{ userId }`, `{ status: "ok" }`). Adopting full JSON:API would require rewriting all existing API contracts. RFC 7231 semantics give us the normative standard we need without imposing a response envelope.

**Alternatives considered:** JSON:API → breaks existing PRD contracts, adds client overhead. gRPC → not feasible given mobile clients and existing HTTP Gateway.

---

### D2 — Folder structure: Hexagonal (ports-and-adapters) for all services

**Decision:** All services follow the hexagonal architecture layout defined in `user-service/FOLDER_STRUCTURE.md`. For Node.js services, the same logical separation applies with JavaScript naming: `adapter/in/web` (controllers), `application/port/in` (use-case interfaces or JSDoc contracts), `application/service` (implementations), `application/port/out` (repository contracts), `adapter/out/data_access` (persistence), `domain` (business entities).

**Rationale:** Using the same structure across all services means any engineer can navigate any service without re-learning the layout. The user-service is already the reference implementation in Spring Boot; replicating the logical structure in Node.js services gives consistency without requiring a Java rewrite of existing services.

**Alternatives considered:** Feature-based flat folders — simpler but does not enforce dependency direction. MVC flat — already in the monolith, causes the coupling we are trying to eliminate.

---

### D3 — Redis scoped to Location Service geo-indexing only

**Decision:** Redis is used in exactly one place: Location Service for `GEOADD`/`GEORADIUS`/`GEORADIUSBYMEMBER` operations on the `driver:geo` key. No other service uses Redis in Phase 1.

**Rationale (removals):**
- **JWT blacklist** → Tokens have short TTL (15 min access / 7 day refresh). In Phase 1, token invalidation on logout is deferred; the blacklist is a Phase 2 concern.
- **Rate limiting** → API Gateway can implement in-memory rate limiting (per-process) for Phase 1. The traffic volume does not require distributed rate limiting.
- **Socket.io adapter** → Location Service is single-instance in Phase 1. In-memory Socket.io adapter is sufficient; Redis adapter becomes necessary only when horizontal scaling is added in Phase 2.
- **Redlock (distributed locks)** → Distributed locking for ride assignment is replaced by MongoDB atomic operations (`findOneAndUpdate` with status filter). No external lock store required.

**Impact:** Location Service depends on Redis (single cluster, geo-index only). All other services have zero Redis dependency.

---

### D4 — PRD files are per-service with module → feature → contract structure

**Decision:** Each service gets its own PRD file. The structure within each PRD is: `## Module: <Name>` → `### Feature: <Name>` → API contract table + Kafka event contract (where applicable). The PRD index (`prd-index.md`) is a flat table mapping `Service | Module | Feature | File | Section Anchor`.

**Rationale:** A flat single-PRD file (as exists in `prd-and-epic-mapping`) becomes unwieldy as the project grows and is hard for AI tooling to navigate with precision. Per-service files allow an AI agent to load only the relevant PRD. The index file gives the agent a routing layer to find the right section without loading all PRDs.

---

### D5 — Cross-service dependencies documented as a dependency matrix + narrative

**Decision:** `cross-service-dependency-map.md` includes: (a) a feature-level dependency matrix table showing which features need which other services, (b) per-dependency narrative explaining the API call or Kafka event required, and (c) a dependency direction graph in Mermaid.

**Rationale:** Without explicit documentation of cross-service dependencies, an engineer building Ride Service may not realise they need to wait for a Location Service geo-query API to be available before ride assignment works. Surfacing these dependencies early prevents blocked sprint work.

---

### D6 — Roadmap sequenced by dependency order, parallelised by team tier

**Decision:** The roadmap is organised in phases where Phase 1 establishes foundations (API Gateway, User Service), Phase 2 parallelises core business services (Ride Service by senior, Location Service by senior), Phase 3 delivers supporting services (Notification by intern pair, Analytics by intern+senior). Each phase specifies which team tier owns each service.

**Rationale:** Interns should build services with well-defined external interfaces and minimal cross-service logic. API Gateway and Notification Service are the best fits. Ride Service and Location Service have the most complex logic and should be owned by senior engineers.

---

## Risks / Trade-offs

**Documents diverge from implementation as code evolves** → Mitigation: PRD files are versioned alongside code. Tasks include a rule that contract changes require a PRD update in the same PR.

**Removing Redis JWT blacklist means logout does not immediately invalidate tokens** → Mitigation: Access tokens have 15-minute TTL; logout removes the client-side token. This is an accepted Phase 1 trade-off. JWT blacklist is a Phase 2 item.

**Hexagonal structure overhead for simple services (API Gateway, Notification)** → Mitigation: For services with one or two modules, the hexagonal structure has only 3-4 files per module but keeps the pattern consistent. The overhead is minimal.

**Per-service PRDs become the new source of truth, potentially diverging from `prd-and-epic-mapping`** → Mitigation: Per-service PRDs are derived from `prd-and-epic-mapping/prd.md`. The original PRD remains the authoritative source for acceptance criteria. Per-service PRDs add module/feature breakdown on top.

**Redis geo-index single point of failure for ride assignment** → Mitigation: Location Service owns the Redis geo-index. If Redis is unavailable, driver assignment falls back to a MongoDB geospatial query (2dsphere index on `driverLocationStatuses`). Document the fallback in Location Service PRD.

---

## Migration Plan

No code migration is required — this change produces documentation only. The output documents are written to `openspec/changes/platform-standards-and-prd/specs/`:

1. Write `api-protocol/spec.md` — HTTP standard
2. Write `folder-structure-standard/spec.md` — folder layout
3. Write `prd-<service>/spec.md` for each of 6 services
4. Write `prd-index/spec.md` — knowledge graph index
5. Write `cross-service-dependency-map/spec.md` — dependency matrix
6. Write `roadmap/spec.md` — delivery plan

Engineers reference these docs from their service repos. No deployment, rollback, or database migration is required.

---

## Open Questions

- **OQ-1:** Should the API standard mandate API versioning via URL prefix (`/api/v1/`) or via `Accept` header? Current PRD uses unversioned paths (`/api/auth/register`). Recommend URL prefix, deferred to Phase 2 unless a breaking change occurs in Phase 1.
- **OQ-2:** Notification Service has no DB in Phase 1 (logs only). Should a `notifications` MongoDB collection be created for delivery receipts? Deferred to Phase 2; the PRD should note this as a Phase 2 consideration.
