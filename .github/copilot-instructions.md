<!-- This file is generated/updated by an AI assistant. Please review and adjust if needed. -->
# Copilot / AI agent instructions for spring-modular-monolith

This repository is an example e‑commerce application built as a Modular Monolith using Spring Modulith.
Keep instructions short and targeted — the goal is to make an automated coding agent productive immediately.

Key facts (brief)
- Language: Java 24, build system: Maven (wrapper `./mvnw`).
- Frameworks: Spring Boot 3.x, Spring Modulith, Spring Data JPA, Flyway, RabbitMQ (AMQP), Thymeleaf web UI.
- Modules live under `src/main/java/com/sivalabs/bookstore/*` (e.g. `catalog`, `orders`, `inventory`, `notifications`, `common`).

Big-picture architecture and intent
- Modular Monolith: each domain is implemented as an independent module (package) with module boundaries declared via Spring Modulith. See `src/main/java/com/sivalabs/bookstore/*` for modules.
- Data isolation: each module manages its own schema (`src/main/resources/db/migration` contains Flyway migrations per schema: `V2__catalog_*.sql`, `V4__orders_*.sql`, etc.).
- Event-driven where possible: modules publish domain events via Spring Modulith events and those are bridged to AMQP (RabbitMQ). Example: Orders publishes `OrderCreatedEvent` → consumed by `inventory` and `notifications` modules.
- Public APIs: modules expose package-level public APIs (e.g. `catalog/ProductApi.java`) for direct invocation when necessary (Orders calls Catalog API to validate products).

Where to look for examples
- Application bootstrap: `src/main/java/com/sivalabs/bookstore/BookStoreApplication.java` and `ApplicationProperties.java` (configuration properties record shapes).
- Module API example: `src/main/java/com/sivalabs/bookstore/catalog/ProductApi.java` and `ProductDto.java`.
- Security and config: `src/main/java/com/sivalabs/bookstore/config/*` (JWT, web security, OpenAPI). Respect these when adding endpoints or wiring components.
- Flyway migrations: `src/main/resources/db/migration` — use these to understand DB schema boundaries and seed data.

Build / test / run (developer workflows)
- Common commands (preferred via `task` runner as described in README):
  - Run tests: `task test` (wrapper around `mvnw` and Testcontainers). If `task` is not available use `./mvnw test`.
  - Format code: `task format` (runs Spotless via Maven). Equivalent: `./mvnw spotless:apply`.
  - Build image: `task build_image` (invokes Spring Boot image build). Equivalent: `./mvnw spring-boot:build-image`.
  - Run app locally: `task start` (docker compose) or `./mvnw spring-boot:run` for local JVM run. App exposes Actuator and Modulith endpoints: `/actuator` and `/actuator/modulith`.
- Tests use Testcontainers and spring-modulith test support. Expect tests to spin up Postgres/RabbitMQ unless you run unit slices.

Conventions and patterns specific to this repo
- Modules are packages under `com.sivalabs.bookstore` — prefer adding new domain code as a new package/module, not by scattering across packages.
- Public API objects are explicit (e.g., `ProductApi`, `ProductDto`) — use these API classes to call across modules instead of directly depending on persistence classes.
- Events: use domain event classes and modulith's event publishing; do not directly publish raw AMQP messages. The project bridges modulith events to AMQP (see `config/RabbitMQConfig.java`).
- Configuration: `ApplicationProperties` is a top-level record mapped from `app.*` — when adding configuration fields, add immutable record fields and update docs.
- Database: migrations per domain live in `resources/db/migration` and are applied via Flyway. Keep schema-specific SQL files grouped by their focal module.
- Formatting: Spotless with palantir-java-format is enforced. Run `task format` before commits.

Integration points and external dependencies
- Database: Postgres. See `pom.xml` dependencies and SQL migrations in `src/main/resources/db/migration`.
- Message broker: RabbitMQ (AMQP). Runtime bridge is configured; test dependencies include `rabbitmq` Testcontainers.
- Observability: Micrometer, OpenTelemetry → Zipkin exporter is included. Actuator endpoints enabled; `spring-modulith-actuator` provides module metadata at `/actuator/modulith`.

How to make safe changes (practical rules for an AI agent)
- Prefer small, local changes. If changing cross-cutting behavior (security, application.properties shape, Flyway migration), add clear tests and update `ApplicationProperties`/docs.
- When adding endpoints, update OpenAPI via annotations/config and place controllers inside the appropriate domain package (e.g., `catalog/web`), and write an integration test under `test/java/com/sivalabs/bookstore/<module>`.
- Respect the modulith boundaries: if you need data from another module, first prefer using the module's public API (e.g., `catalog/ProductApi`) or publish/consume domain events.
- Do not add new runtime dependencies without a clear reason — document the need in the code comments and tests.

Examples to reference in PRs
- Validating an order against catalog: see `orders` module calls to `catalog/ProductApi` (search for usages of `ProductApi`).
- Event consumption: look for an `@EventListener` or modulith event handler inside `inventory`/`notifications`.

Files to update when making infra or API changes
- `README.md` — update run/dev instructions if you change the `task` targets or ports.
- `src/main/resources/db/migration` — add Flyway migrations for any schema changes.
- `ApplicationProperties.java` — update if adding new `app.*` configuration keys.
- `pom.xml` — update only when adding test/runtime libraries; keep Java version consistent (`<java.version>24</java.version>`).

If you need more context
- Ask for which module or feature to change and the preferred style (API vs events). Point to the specific package and I'll open those files.

End of instructions — ask the human for any missing or outdated guidance you'd like me to include.
