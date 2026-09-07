# Linktic Java Backend Assessment

Java microservices assessment for a purchase flow: a product-catalog API, an inventory API, and purchase history behind an NGINX API gateway. The services run with isolated PostgreSQL databases through Docker Compose.

![License: AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0-blue) ![Java](https://img.shields.io/badge/backend-Java%20%2B%20Spring%20Boot-6DB33F) ![Database](https://img.shields.io/badge/database-PostgreSQL-4169E1)

![Architecture diagram](Docs/Diagrama-Arquitectura.svg)

## Contents

- [Problem and scope](#problem-and-scope)
- [Architecture and decisions](#architecture-and-decisions)
- [Purchase flow and consistency](#purchase-flow-and-consistency)
- [API, security, and errors](#api-security-and-errors)
- [Resilience and observability](#resilience-and-observability)
- [Testing and delivery](#testing-and-delivery)
- [Repository layout](#repository-layout)
- [Running with Docker Compose](#running-with-docker-compose)
- [Evolution path](#evolution-path)
- [License](#license)

## Problem and scope

The assessment implements a catalog and inventory domain with a purchase operation. A client can access the services through a common gateway, check product and stock data, and create purchases while preserving a traceable history of successful transactions.

The implementation demonstrates a modular microservice design rather than a full production platform. It prioritizes clear service ownership, local reproducibility, explicit failure handling, and an architecture that can evolve without merging all concerns into one application.

## Architecture and decisions

NGINX is the public entry point and routes requests to two Spring Boot services:

| Component | Ownership | Design rationale |
| --- | --- | --- |
| NGINX API gateway | Public routing and Basic Auth. | Centralizes external routes and leaves services on the internal Compose network. |
| Products service | Product catalog and product details. | Keeps catalog data and its API independent from inventory concerns. |
| Inventory service | Stock, purchase orchestration, and purchase history. | Owns stock changes and coordinates the purchase lifecycle. |
| PostgreSQL databases | One database per service. | Preserves service boundaries and avoids a shared schema becoming an implicit coupling point. |
| Docker Compose | Services, networks, and volumes. | Reproduces the complete topology locally with persistent database volumes. |

The product and inventory services communicate over HTTP. Each service owns its data and exposes a focused API, while the gateway protects and routes the public entry points.

### Why `/compra` belongs to inventory

The `/compra` endpoint is implemented by the inventory service because it is the service that verifies and modifies stock. This gives one component responsibility for the transaction sequence and avoids putting stock orchestration inside the product-catalog service, where it would violate the single-responsibility boundary.

### Service and database separation

PostgreSQL is used because the domain is relational and needs transactional consistency within each service. A separate database per service prevents direct cross-service table dependencies and allows each service to evolve its own schema and deployment cadence.

The trade-off is that a purchase cannot rely on one database transaction spanning all services. The design therefore uses explicit orchestration and compensating actions rather than hiding distributed consistency behind a shared database.

## Purchase flow and consistency

The inventory service orchestrates the purchase as a sequential saga:

1. Verify that the product exists in local inventory and that sufficient stock is available.
2. Call the products service to confirm the catalog data.
3. Persist the consolidated purchase record in purchase history.
4. Decrement the available inventory.
5. If the inventory adjustment fails after the purchase record is created, compensate by removing that purchase record and return a failed response.

Validating stock locally before the remote catalog call avoids unnecessary network traffic for a purchase that cannot succeed. Persisting the history before decrementing stock creates an auditable transaction boundary; the compensation path prevents a successful-looking purchase from remaining recorded when inventory was not actually updated.

### Why an orchestrated saga

The inventory service is the natural orchestrator because it owns the stock decision and sees the complete purchase sequence. An event-choreographed saga would remove the central coordinator but make this short, ordered workflow harder to inspect and compensate. Two-phase commit would provide stronger atomicity but would add a global coordinator, cross-database coupling, locks, and operational complexity that are disproportionate to this assessment.

The current design makes each step, failure point, and compensation explicit. It is suitable for a sequential workflow with a small number of participants and can later evolve to asynchronous events if throughput or integration needs require it.

## API, security, and errors

Swagger UI is served by each service:

- `http://localhost:8080/ms-productos/swagger-ui.html`
- `http://localhost:8080/ms-inventario/swagger-ui.html`

The Postman collection is available at [`Docs/Prueba Tecnica Java.postman_collection.json`](Docs/Prueba%20Tecnica%20Java.postman_collection.json). It includes requests organized by service, examples for common success and failure responses, and local/Docker environments.

### Authentication boundary

The gateway protects the exposed routes with Basic Auth. The repository's local test credentials are `admin` / `admin123`.

Those credentials are only appropriate for local assessment use. They must not be exposed to a public network or reused in a deployment. A production design should move credentials into a secret store, use TLS, centralize authentication with OAuth2 or JWT, and introduce authorization roles and scopes.

### Response and error contract

The services use DTOs to separate transport data from domain entities and return a consistent response structure for success and failure paths. A global `@ControllerAdvice` captures validation and deserialization errors centrally, allowing clients and tests to parse failures predictably.

```json
{
  "mensaje": "Error de formato en los campos",
  "data": {
    "nombre": "El nombre es obligatorio"
  }
}
```

## Resilience and observability

The inventory service applies timeouts and retries when it calls the products service. This prevents a temporary downstream failure from blocking the request indefinitely and gives transient network errors a bounded retry path.

Structured application logs record severity, timestamp, source service, module, message, and request-error context. The global exception handler logs validation and deserialization failures consistently, while successful operations are also logged for traceability.

![Structured logs](Docs/Logs.png)

The current logs are ready to be collected by a central system, but the repository does not include an ELK, Prometheus, or Grafana deployment. A production environment should add correlation IDs at the gateway, metrics for downstream calls and retries, and alerting for failed purchase compensations.

## Testing and delivery

Both Spring Boot services include Maven wrappers and test source trees. Run each service independently:

```bash
cd ms-productos
./mvnw test
```

```bash
cd ms-inventario
./mvnw test
```

The repository does not include CI/CD, a coverage gate, or configured linters such as Checkstyle, PMD, or SpotBugs. These are the highest-value delivery additions for a collaborative or production workflow:

- Run Maven tests, static analysis, and dependency checks on every push and pull request.
- Add contract and integration tests for the gateway and purchase saga, including compensation scenarios.
- Publish a versioned container image only after the quality gate passes.
- Move runtime configuration and credentials to environment variables or a secret manager.

## Repository layout

- `ms-productos/`: product catalog Spring Boot service, Maven wrapper, and test sources.
- `ms-inventario/`: inventory, purchase-history, and orchestration Spring Boot service.
- `nginx/`: API gateway configuration.
- `Docs/`: architecture diagram, requirements, structured-log evidence, and Postman collection.
- `docker-compose.yml`: local multi-service topology and persistent dependencies.

The code follows a layered structure within each service: controllers adapt HTTP requests, services contain application logic, models carry entities and DTOs, repositories manage persistence, and configuration centralizes security and application settings. This keeps a later migration toward stricter hexagonal boundaries possible without throwing away the existing organization.

## Running with Docker Compose

```bash
git clone https://github.com/juanhdzma/linktic-senior-backend-test.git
cd linktic-senior-backend-test
docker-compose up --build
```

Wait for the products, inventory, PostgreSQL, and gateway containers to finish starting. Then use Swagger UI or import the Postman collection. The gateway endpoints are available under `http://localhost:8080/ms-productos` and `http://localhost:8080/ms-inventario`.

## Evolution path

The current synchronous design is deliberately small. Future evolution can be driven by measured needs:

- Replace synchronous internal calls with Kafka or RabbitMQ events where asynchronous throughput matters.
- Extract purchase history into its own service when its scale or reporting workload justifies independent ownership.
- Add Redis only after measuring repeated-read pressure that a cache can meaningfully reduce.
- Add circuit breakers and exponential backoff around downstream dependencies.
- Introduce centralized OAuth2/JWT authentication, authorization, and secret management.
- Add metrics, traces, dashboards, and alerts before scaling the number of services.

## License

[AGPL-3.0](LICENSE)
