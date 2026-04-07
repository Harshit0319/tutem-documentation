## ADDED Requirements

### Requirement: All services follow hexagonal (ports-and-adapters) folder layout
Every TUTEM service SHALL use the folder structure below. This is derived exactly from `tutem_services/user-service/FOLDER_STRUCTURE.md`. All services are Java 17 / Spring Boot 3.

```
<service>/
├─ build.gradle
├─ gradle.properties
├─ settings.gradle
├─ SETUP.md
└─ src/
   ├─ main/
   │  ├─ java/com/tutem/<service>/
   │  │  ├─ modules/
   │  │  │  └─ <module>/
   │  │  │     ├─ adapter/
   │  │  │     │  ├─ in/web/              ← @RestController, request/response DTOs,
   │  │  │     │  │                          @Valid input annotations, HTTP status mapping
   │  │  │     │  └─ out/data_access/
   │  │  │     │     └─ mongo/            ← @Document models, MongoRepository / MongoTemplate,
   │  │  │     │                             mapping between persistence document and domain entity
   │  │  │     ├─ application/
   │  │  │     │  ├─ port/in/             ← Use-case interfaces (@UseCase)
   │  │  │     │  ├─ port/out/            ← Repository and external dependency abstractions
   │  │  │     │  └─ service/             ← @Service implementations, workflow orchestration
   │  │  │     └─ domain/
   │  │  │        └─ entity/              ← Plain Java classes: business entities, value objects,
   │  │  │                                   domain rules, domain exceptions
   │  │  ├─ shared/
   │  │  │  ├─ constants/                 ← String constants, enum definitions, config keys
   │  │  │  ├─ errors/                    ← Custom exception classes, error code enums
   │  │  │  └─ logging/                   ← Logger factory, structured log helpers
   │  │  └─ config/                       ← @Configuration classes, bean definitions, app wiring
   │  └─ resources/
   │     ├─ application.yml
   │     ├─ application-local.yml
   │     ├─ application-dev.yml
   │     ├─ application-stage.yml
   │     └─ application-prod.yml
   └─ test/
      └─ java/com/tutem/<service>/        ← Unit tests (domain, service), integration tests (adapters)
```

#### Scenario: New API endpoint placement
- **WHEN** a developer adds a new REST endpoint
- **THEN** it SHALL be placed in `modules/<module>/adapter/in/web/` as a `@RestController` class

#### Scenario: New use-case placement
- **WHEN** a developer adds a new business operation
- **THEN** the interface SHALL be in `modules/<module>/application/port/in/` and the `@Service` implementation in `modules/<module>/application/service/`

#### Scenario: Database code placement
- **WHEN** a developer adds persistence logic
- **THEN** the `@Document` model and `MongoRepository` SHALL be in `modules/<module>/adapter/out/data_access/mongo/`; domain entities in `domain/entity/` SHALL NOT carry `@Document` or any Spring Data annotations

---

### Requirement: Dependency direction is strictly enforced
Within a module, dependencies SHALL only flow in one direction:

```
adapter/in → application/port/in → application/service → application/port/out → adapter/out
```

`domain` is independent — it SHALL NOT import from adapter or infrastructure packages.

#### Scenario: Domain entity has no Spring annotations
- **WHEN** a domain entity class is inspected
- **THEN** it SHALL NOT carry `@Document`, `@Entity`, `@RestController`, `@GetMapping`, or any Spring/Mongo annotation

#### Scenario: Service layer has no HTTP concerns
- **WHEN** an `application/service/` class is inspected
- **THEN** it SHALL NOT import `HttpServletRequest`, `ResponseEntity`, or any `org.springframework.web` type; HTTP concerns belong in the adapter layer

---

### Requirement: Naming conventions
All classes SHALL follow the naming conventions below.

| Layer | Class name pattern | Example |
|-------|--------------------|---------|
| Controller (`adapter/in/web`) | `<Module>Controller` | `UserController` |
| Use-case port (`application/port/in`) | `<Action><Module>UseCase` | `CreateUserUseCase` |
| Service (`application/service`) | `<Action><Module>Service` | `CreateUserService` |
| Repository port (`application/port/out`) | `<Module>RepositoryPort` | `UserRepositoryPort` |
| Repository adapter (`adapter/out/data_access/mongo`) | `<Module>RepositoryAdapter` | `UserRepositoryAdapter` |
| Domain entity (`domain/entity`) | `<Module>` | `User` |
| Persistence document | `<Module>Document` | `UserDocument` |

#### Scenario: Controller naming
- **WHEN** a new controller is created for the `ride` module
- **THEN** it SHALL be named `RideController`

---

### Requirement: Shared code placement
Code used by more than one feature module SHALL be placed in `shared/`.

- `shared/constants/` — string constants, enums, configuration keys
- `shared/errors/` — custom exception classes, error code definitions
- `shared/logging/` — logger factory, log format helpers

Cross-module business logic SHALL NOT be placed in `shared/`. If two modules share business logic, extract a third module.

#### Scenario: Shared error class
- **WHEN** a custom exception is needed in more than one module
- **THEN** it SHALL be defined in `shared/errors/` and imported by each module

---

### Requirement: Environment configuration profiles
Each service SHALL have a Spring profile config file for each deployment environment.

Required profiles and files:

| Profile | File | Purpose |
|---------|------|---------|
| (base) | `application.yml` | Common config shared across all profiles |
| `local` | `application-local.yml` | Local development; local MongoDB instance |
| `dev` | `application-dev.yml` | Shared dev environment |
| `stage` | `application-stage.yml` | Staging / QA environment |
| `prod` | `application-prod.yml` | Production; no debug logging |

Active profile is selected via `--spring.profiles.active=<profile>` at startup or the `SPRING_PROFILES_ACTIVE` environment variable.

#### Scenario: Local config overrides base
- **WHEN** the service starts with `--spring.profiles.active=local`
- **THEN** values in `application-local.yml` override `application.yml`; no production MongoDB URI is used

#### Scenario: Prod profile disables debug logging
- **WHEN** the service starts with `--spring.profiles.active=prod`
- **THEN** log level SHALL be `INFO` or higher; no `DEBUG` or `TRACE` output

---

### Requirement: Quick placement reference
The following table SHALL be used as a quick lookup when deciding where to place new code.

| What you're adding | Where it goes |
|--------------------|---------------|
| New REST endpoint | `modules/<module>/adapter/in/web/` |
| New use-case interface | `modules/<module>/application/port/in/` |
| New use-case implementation | `modules/<module>/application/service/` |
| New repository contract | `modules/<module>/application/port/out/` |
| New Mongo adapter / document | `modules/<module>/adapter/out/data_access/mongo/` |
| New business entity or rule | `modules/<module>/domain/entity/` |
| New Spring bean / config | `config/` |
| New profile config | `resources/application-<profile>.yml` |
| Shared constant / error / logger | `shared/<constants|errors|logging>/` |
