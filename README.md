# CommerceHub Microservice

A focused backend microservice for order processing and inventory management built with .NET 8, MongoDB, and RabbitMQ.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | .NET 8 Web API |
| Database | MongoDB 7 |
| Messaging | RabbitMQ 3 (with Management UI) |
| Testing | nUnit 4, Moq, FluentAssertions |
| Containerization | Docker & Docker Compose |

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (includes Docker Compose)
- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) — only needed to run tests locally

---

## Running the Full Environment

A single command starts the API, MongoDB, and RabbitMQ:

```bash
docker-compose up --build
```

Wait for all three health checks to pass (roughly 30–45 seconds on first run). You will see log output confirming:
- MongoDB is ready
- RabbitMQ is ready
- The API has started and connected to both

| Service | URL |
|---|---|
| API + Swagger UI | http://localhost:8080 |
| RabbitMQ Management | http://localhost:15672 (guest / guest) |
| MongoDB | mongodb://localhost:27017 |

To stop and clean up:

```bash
docker-compose down
```

To remove persisted data volumes as well:

```bash
docker-compose down -v
```

---

## API Endpoints

### POST `/api/orders/checkout`

Process a new order. Verifies stock, atomically decrements inventory, creates the order record, and publishes an `OrderCreated` event to RabbitMQ.

```json
POST /api/orders/checkout
{
  "customerId": "cust-123",
  "items": [
    { "productId": "64f1a2b3c4d5e6f7a8b9c0d1", "quantity": 2 }
  ]
}
```

Responses:
- `201 Created` — order created, full order object returned
- `400 Bad Request` — validation failure (e.g. quantity < 1)
- `409 Conflict` — product out of stock or does not exist

---

### GET `/api/orders/{id}`

Retrieve an order by ID.

Responses:
- `200 OK` — order object
- `404 Not Found` — order does not exist

---

### PUT `/api/orders/{id}`

Update an order's status or customer. Idempotent. Blocked if the order is already `Shipped`.

```json
PUT /api/orders/64f1a2b3c4d5e6f7a8b9c0d1
{
  "status": "Processing"
}
```

Responses:
- `200 OK` — updated order
- `404 Not Found` — order does not exist
- `409 Conflict` — order is already Shipped

---

### PATCH `/api/products/{id}/stock`

Directly adjust product stock. Atomic operation — safe under concurrent load. Blocked if the result would go below zero.

```json
PATCH /api/products/64f1a2b3c4d5e6f7a8b9c0d1/stock
{
  "delta": -5
}
```

Responses:
- `200 OK` — updated product with new stock level
- `404 Not Found` — product does not exist
- `409 Conflict` — delta would produce negative stock

---

## Running Tests

Run the full nUnit test suite locally (no Docker required — all dependencies are mocked):

```bash
cd src/CommerceHub.Tests
dotnet test --verbosity normal
```

Or from the solution root:

```bash
dotnet test CommerceHub.sln --verbosity normal
```

Test coverage areas:

| Area | File |
|---|---|
| Validation (negative quantity, zero quantity) | `OrderTests/OrderServiceTests.cs` |
| Atomic stock decrement logic | `OrderTests/OrderServiceTests.cs` |
| Event emission (mock publisher verification) | `EventTests/EventEmissionTests.cs` |
| Stock adjustment edge cases and concurrency | `ProductTests/ProductServiceTests.cs` |

---

## Seeding Test Data

Insert a product directly into MongoDB to test the checkout flow:

```bash
docker exec -it commercehub-mongo mongosh
```

```js
use CommerceHub

db.Products.insertOne({
  name: "Widget Pro",
  description: "A high quality widget",
  price: 29.99,
  stock: 100,
  createdAt: new Date(),
  updatedAt: new Date()
})
```

Copy the `_id` from the output and use it in your checkout request.

---

## Project Structure

```
CommerceHub/
├── src/
│   ├── CommerceHub.API/
│   │   ├── Controllers/        # OrdersController, ProductsController
│   │   ├── Models/             # Order, Product, DTOs
│   │   ├── Services/           # OrderService, ProductService, RabbitMqPublisher
│   │   ├── Repositories/       # OrderRepository, ProductRepository (MongoDB)
│   │   ├── Events/             # OrderCreatedEvent
│   │   ├── Configuration/      # MongoDbSettings, RabbitMqSettings
│   │   ├── Program.cs
│   │   └── appsettings*.json
│   └── CommerceHub.Tests/
│       ├── OrderTests/         # OrderServiceTests
│       ├── ProductTests/       # ProductServiceTests
│       └── EventTests/         # EventEmissionTests
├── Dockerfile
├── docker-compose.yml
├── README.md
└── developer-log.md
```

---

## Environment Variables

All connection strings are injected via environment variables in Docker Compose. For local development without Docker, edit `appsettings.json` or set:

```bash
MongoDb__ConnectionString=mongodb://localhost:27017
RabbitMq__HostName=localhost
```
