# Linktic Java Backend Assessment

Java microservices assessment for a purchase flow: a product-catalog API, an inventory API, and purchase history behind an NGINX API gateway. The services run with isolated PostgreSQL databases through Docker Compose.

![License: AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0-blue) ![Java](https://img.shields.io/badge/backend-Java%20%2B%20Spring%20Boot-6DB33F) ![Database](https://img.shields.io/badge/database-PostgreSQL-4169E1)

![Architecture diagram](Docs/Diagrama-Arquitectura.svg)

## Contents

- [Architecture](#architecture)
- [Purchase flow](#purchase-flow)
- [API and authentication](#api-and-authentication)
- [Running with Docker Compose](#running-with-docker-compose)
- [Testing](#testing)
- [Repository layout](#repository-layout)
- [License](#license)

## Architecture

NGINX is the public entry point and routes requests to two independent Spring Boot services:

- **Products** owns the product catalog.
- **Inventory** owns stock, orchestrates purchases, and persists completed purchase history.

Each service has its own PostgreSQL database, keeping ownership and failure boundaries explicit. The gateway applies Basic Auth and exposes the public routes while the service-to-service communication stays inside the Compose network.

## Purchase flow

The inventory service coordinates a purchase in this order:

1. Validates local stock.
2. Checks the product through the products service.
3. Persists the purchase history.
4. Updates inventory.

Timeouts and retries protect the internal product lookup. The flow is structured so that failures can be handled with compensating actions instead of leaving inventory and purchase history inconsistent.

## API and authentication

Swagger UI is served by each service:

- `http://localhost:8080/ms-productos/swagger-ui.html`
- `http://localhost:8080/ms-inventario/swagger-ui.html`

The local test credentials are `admin` / `admin123`. The Postman collection is available at [`Docs/Prueba Tecnica Java.postman_collection.json`](Docs/Prueba%20Tecnica%20Java.postman_collection.json).

Responses use a consistent envelope for successful and failed requests. Request logging includes the source service, module, severity, timestamp, and error context to make the purchase flow traceable.

![Structured logs](Docs/Logs.png)

## Running with Docker Compose

```bash
git clone https://github.com/juanhdzma/linktic-senior-backend-test.git
cd linktic-senior-backend-test
docker-compose up --build
```

Wait for the products, inventory, PostgreSQL, and gateway containers to finish starting, then use the Swagger endpoints or import the Postman collection.

## Testing

Each Spring Boot service includes a Maven wrapper and a test source tree. Run its tests independently:

```bash
cd ms-productos
./mvnw test
```

```bash
cd ms-inventario
./mvnw test
```

## Repository layout

```text
ms-productos/    Product catalog microservice
ms-inventario/   Inventory and purchase-history microservice
nginx/           API gateway configuration
Docs/            Architecture, requirements, logs, and Postman collection
docker-compose.yml
```

## License

[AGPL-3.0](LICENSE)
