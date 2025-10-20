# Spring Modular Monolith — Architecture and Communication Analysis

Date: 2025-10-20

This document provides a senior-architect level analysis of the repository’s architecture, modular boundaries, communication styles, and design patterns. It proceeds module-by-module and then synthesizes cross-cutting concerns.

## Overview

- Style: Modular Monolith implemented with Spring Boot 3.x and Spring Modulith.
- Language/Build: Java 24, Maven.
- Domains (modules): catalog, orders, inventory, notifications, users, plus common and config.
- Data isolation: Postgres with a schema-per-module pattern via Flyway migrations.
- Communication:
  - Synchronous in-process via public APIs exposed by modules (e.g., `catalog/ProductApi`).
  - Asynchronous via domain events using Spring Modulith’s event publication and JDBC outbox; externalized to AMQP (RabbitMQ).
  - External interfaces via REST controllers and a Thymeleaf web UI.
- Observability & Ops: Spring Actuator including Modulith endpoints, Micrometer + OpenTelemetry with Zipkin exporter; Testcontainers for integration tests.

## Top-level architecture and runtime

- Entry point: `BookStoreApplication` with `@SpringBootApplication` and `@ConfigurationPropertiesScan`.
- Configuration surface: `ApplicationProperties` defines strongly-typed, immutable configuration for JWT, CORS, and OpenAPI.
- Build: `pom.xml` includes Spring Modulith, JPA, AMQP, OpenAPI, Thymeleaf, and test deps (Testcontainers, Modulith test support). Java 24 is configured.
- Database: `src/main/resources/db/migration` contains Flyway migrations. `V1__create_schemas.sql` creates `catalog`, `orders`, `inventory`, `events` (and later `users`). Each module owns its schema and migrations.
- Events: Modulith JDBC event publication configured to use `events` schema with republish-on-restart enabled.
- Messaging: RabbitMQ declared with `TopicExchange BookStoreExchange`, routing key `orders.new`, queue `new-orders`. Modulith bridges externalized events to AMQP.

## Modularity and boundaries

- Spring Modulith treats each top-level package under `com.sivalabs.bookstore` as a module. Key module annotations:
  - `orders/package-info.java`: `@ApplicationModule(allowedDependencies = {"catalog", "users"})` explicitly constrains dependencies.
  - `common/package-info.java`: `@ApplicationModule(type = ApplicationModule.Type.OPEN)` to allow shared models/utilities to be accessed.
- Public module APIs are defined as Spring components in the module root (e.g., `catalog/ProductApi.java`, `orders/OrdersApi.java`). Other modules should depend on these APIs instead of persistence classes to preserve encapsulation.
- Value Objects are modeled as Java records and embedded where appropriate.

## Module-by-module analysis

### 1) Catalog

- Purpose: Product catalog read model and lookup service.
- API: `catalog/ProductApi` exposes `Optional<ProductDto> getByCode(String code)` for internal calls from other modules (e.g., Orders validation).
- Web: `catalog/web/ProductRestController` exposes public endpoints:
  - `GET /api/products` returns `PagedResult<ProductDto>` with page metadata (via `common/models/PagedResult`).
  - `GET /api/products/{code}` returns a single product.
- Domain: `ProductService`, `ProductRepository`, `ProductEntity`, `ProductMapper`.
- Data: Flyway `V2__catalog_create_products_table.sql` creates `catalog.products` with numeric `price`. `V3__catalog_add_books_data.sql` seeds data (in repo).
- Patterns:
  - Service + Repository + DTO Mapper.
  - Public API acts as a module boundary (internal port).
  - Pagination abstraction in `common` decouples controllers from Spring Data Page.

### 2) Orders

- Purpose: Manage orders end-to-end, publish domain events, and enforce user scoping.
- Boundaries: Declared in `orders/package-info.java` to depend only on `catalog` and `users`.
- Entry points:
  - REST: `orders/web/OrderRestController` exposes:
    - `POST /api/orders` (create): Requires JWT; uses `UserContextUtils` to bind `userId`. Validates product with `ProductServiceClient` (sync call to Catalog API) before persisting. Publishes `OrderCreatedEvent` upon success.
    - `GET /api/orders/{orderNumber}` (per-user view).
    - `GET /api/orders` (list per user).
  - Programmatic API: `orders/OrdersApi` exposes create and read operations for in-process calls.
- Domain: `OrderService` (business + event publishing), `OrderEntity` with embedded `Customer` and `OrderItem` value objects, `OrderRepository` with fetch join queries, `OrderStatus` enum.
- Events: `orders/domain/models/OrderCreatedEvent` is annotated `@Externalized("BookStoreExchange::orders.new")`, ensuring:
  - In-process event dispatch to listeners within the monolith (Inventory, Notifications).
  - Externalization to RabbitMQ exchange `BookStoreExchange` with routing key `orders.new` via Modulith AMQP bridge.
- Data: `V4__orders_create_orders_table.sql` and `V9__orders_add_user_id_to_orders_table.sql` manage schema. Note `product_price` is stored as `text` (potential improvement: numeric).
- Patterns:
  - In-process sync validation through Catalog public API.
  - Transactional create with event publication (backed by Modulith JDBC outbox for reliability).
  - Explicit module dependencies encourage clean architecture boundaries.

### 3) Inventory

- Purpose: Maintain stock levels in response to order creation.
- Event handling: `inventory/OrderEventInventoryHandler` uses `@ApplicationModuleListener` to handle `OrderCreatedEvent`; calls `InventoryService.decreaseStockLevel`.
- Service: `InventoryService` performs stock decrements; logs actions; warns on missing product code; reads via `getStockLevel`.
- Data: `V6__inventory_create_inventory_table.sql` creates `inventory.inventory` and seeds initial stock.
- Testing: `InventoryIntegrationTests` uses Modulith `Scenario` to publish an `OrderCreatedEvent` and waits for the stock level to reach the expected value—illustrating eventual consistency testing.
- Patterns:
  - Event-driven reaction, idempotency not yet explicit (consider upsert and negative stock guard).
  - Persistence through repository (not shown in this doc but present).

### 4) Notifications

- Purpose: Side-effects on order creation (e.g., email, SMS). Current implementation logs receipt.
- Event handling: `notifications/OrderEventNotificationHandler` listens to `OrderCreatedEvent` with `@ApplicationModuleListener`.
- Future evolution: Integrate with an email/SMS provider or push via AMQP; remain decoupled from Orders.

### 5) Users

- Purpose: Authentication, user management, JWT issuance.
- REST:
  - `POST /api/login`: Authenticates credentials using `AuthenticationManager`, issues a JWT using `JwtTokenHelper`. Response includes token, expiry, and user info.
  - `POST /api/users`: Registers new users; persists with role `ROLE_USER` by default.
- Domain: `UserEntity`, `UserRepository`, `UserService`, `SecurityUserDetailsService`, role model `Role`, DTOs and mappers.
- JWT:
  - `JwtSecurityConfig` provides `JwtDecoder` and `JwtEncoder` from RSA keys configured via `ApplicationProperties` (PEM files in `resources/certs`).
  - `UserContextUtils.getCurrentUserIdOrThrow()` parses `user_id` claim from JWT to scope /api requests.
- Data: `V7__create_users_table.sql` and `V8__users_add_users_data.sql` seed data.

### 6) Common and Config

- `common` module: Marked OPEN; contains `PagedResult<T>` to standardize pagination responses.
- `config` module: Cross-cutting infrastructure.
  - Security:
    - `ApiSecurityConfig` (`@Order(1)`): Applies to `/api/**` with stateless JWT resource server. Public endpoints: `/api/login`, `POST /api/users`, `GET /api/products/**`. Others require JWT.
    - `WebSecurityConfig` (`@Order(2)`): Applies to non-API paths with form login, role-based access for `/admin/**`, and static asset/public paths.
    - `CommonSecurityConfig`: `PasswordEncoder`, `AuthenticationManager` via DAO provider, and `RoleHierarchy` based on `users.domain.Role` hierarchy.
  - OpenAPI: `OpenApiConfig` builds OpenAPI with bearer JWT scheme; metadata from `ApplicationProperties`.
  - Web MVC: `WebMvcConfig` sets CORS using `app.cors.*` properties (path pattern defaults to `/api/**`).
  - Messaging: `RabbitMQConfig` defines exchange/queue/binding and JSON message conversion; aligns with `@Externalized("BookStoreExchange::orders.new")` on events.

## Communication styles and protocols

- Intra-module synchronous calls:
  - Orders → Catalog via `ProductApi` for product validation in `ProductServiceClient`. This preserves module encapsulation and avoids direct repository/entity coupling.
- Asynchronous events (reliable, eventually consistent):
  - Orders publishes `OrderCreatedEvent` during `OrderService.createOrder`.
  - Inventory and Notifications consume via `@ApplicationModuleListener`.
  - Event is externalized to AMQP (RabbitMQ) using Modulith AMQP bridge; exchange/routing configured in `RabbitMQConfig` matches `@Externalized` metadata.
  - Modulith JDBC event publication leverages the `events` schema for outbox-style durability and supports republishing on restart.
- External interfaces:
  - REST endpoints under `/api/**` secured by JWT (resource server).
  - Web UI via Thymeleaf controllers for human-facing flows (cart, orders pages), secured by form login.

## Design and architectural patterns

- Modular Monolith via Spring Modulith: explicit module boundaries and dependency verification (see `ModularityTests`).
- Layered within modules: Controller → Service → Repository; public APIs at module root act as ports for cross-module calls.
- Event-driven architecture for inter-module workflows; events model business facts and drive downstream state changes.
- DTO mappers at module boundaries to decouple external shape from persistence.
- Value Objects as Java records (`Customer`, `OrderItem`) embedded in JPA entities.
- Security split: separate filter chains for API vs Web; JWT/OAuth2 Resource Server for APIs, username/password form login for web.
- Observability: Micrometer/OTel, Zipkin exporter, Spring Actuator including `/actuator/modulith` for module metadata.

## Testing and verification

- Modularity verification: `ModularityTests#verifiesModularStructure()` uses Spring Modulith to verify dependencies; `Documenter` can generate module docs.
- API and domain tests:
  - `ProductRestControllerTests`: integration tests using `ApplicationModuleTest` with Testcontainers; asserts pagination + item retrieval.
  - `OrderRestControllerTests`: mocks `ProductApi` to focus on orders; uses JWT issuance to authenticate requests; asserts event publication with `AssertablePublishedEvents`.
  - `InventoryIntegrationTests`: uses `Scenario` to publish `OrderCreatedEvent` and waits for stock update.

## Transaction and consistency model

- Orders’ `createOrder` is transactional; upon persist, publishes `OrderCreatedEvent`. With Modulith JDBC, events are stored and dispatched reliably, enabling eventual consistency for downstream modules and out-of-process consumers via AMQP.
- Read models (e.g., Inventory stock) eventually reflect the Order state post-commit; tests demonstrate this with `Scenario.andWaitForStateChange`.

## Security model

- APIs: OAuth2 Resource Server validating JWTs signed by local RSA keys; `UserContextUtils` ensures user-specific resource access by extracting `user_id` claim.
- Web UI: session-based form login with role checks; static assets and public pages are whitelisted.
- Role hierarchy configured using `Role.getRoleHierarchy()` to allow ADMIN to inherit USER permissions.

## Operational considerations

- Configuration: `application.properties` supplies DB, RabbitMQ, Modulith events, and observability settings. CORS is configured for `/api/**`.
- Docker compose (`compose.yml`) and Taskfile/README (not detailed here) facilitate local development.
- Actuator: set to expose all endpoints for dev; includes health probes, tracing, and Modulith endpoints.

## Strengths and opportunities

- Strengths:
  - Clear module boundaries and public APIs enforce encapsulation.
  - Reliable event publication with JDBC outbox and AMQP externalization.
  - Dual security model fits both API consumers and human web users.
  - Solid test strategy using Modulith test support and Testcontainers.
- Opportunities:
  - Orders DB: consider storing `product_price` as `numeric` instead of `text` for stronger typing.
  - Inventory idempotency and min-bound checks to prevent negative stock; consider correlating by orderNumber to ensure exactly-once semantics.
  - Notifications: extend to real channels (email/SMS) and consider retries/backoff.
  - Catalog/ProductApi: introduce versioning or richer DTOs if cross-module needs grow.
  - Observability: expand tracing across event boundaries (propagate trace context in events if needed).

## Quick reference (files)

- Bootstrap: `BookStoreApplication.java`, `ApplicationProperties.java`.
- Security: `config/ApiSecurityConfig.java`, `config/WebSecurityConfig.java`, `config/JwtSecurityConfig.java`, `config/CommonSecurityConfig.java`.
- Events/Messaging: `orders/domain/models/OrderCreatedEvent.java`, `config/RabbitMQConfig.java`.
- Catalog: `catalog/ProductApi.java`, `catalog/web/ProductRestController.java`, `catalog/domain/*`.
- Orders: `orders/web/OrderRestController.java`, `orders/OrdersApi.java`, `orders/domain/*`, `orders/mappers/OrderMapper.java`.
- Inventory: `inventory/OrderEventInventoryHandler.java`, `inventory/InventoryService.java`, `inventory/*`.
- Notifications: `notifications/OrderEventNotificationHandler.java`.
- Users: `users/web/UserRestController.java`, `users/domain/*`, `users/UserContextUtils.java`.
- Common: `common/models/PagedResult.java`.
- Flyway: `src/main/resources/db/migration/*`.
- Tests: `src/test/java/com/sivalabs/bookstore/*` including `ModularityTests` and module-specific tests.

---

End of analysis.