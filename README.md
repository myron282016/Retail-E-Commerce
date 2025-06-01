# Order & Inventory Microservices (Event-Driven System)

This project demonstrates a simple yet realistic event-driven backend architecture using Java Spring Boot microservices. It showcases how to:

- Build decoupled services using **RabbitMQ** as the message broker.
- Handle **multiple order types** and validate data.
- Implement **idempotency** using **Caffeine Cache**.
- Ensure fault tolerance using **retry logic** and **dead-letter queues (DLQ)**.
- Operate **without circuit breakers** due to the asynchronous nature of the system.

---

## 📦 Services Overview

### 1. Order Service
- Accepts `POST` requests from clients for order submissions.
- Supports multiple order types (e.g., `IN_STORE`, `DIGITAL`).
- Validates input and transforms payloads depending on content type.
- Publishes processed orders to a **RabbitMQ exchange** for downstream services.

### 2. Inventory Service
- Subscribes to order events from RabbitMQ.
- Uses **Caffeine Cache** for idempotent processing — avoids duplicate updates.
- Updates local inventory and maintains its own internal order record (separate DB).
- Simulates business and technical error handling.

---

## 🧱 Architecture
    +-----------+        POST /orders        +------------------+
    |  Client   |  ----------------------->  |   Order Service   |
    +-----------+                           +---------+--------+
                                                   |
                                             Validates + Transforms
                                                   |
                                             Publishes to RabbitMQ
                                                   ↓
                                        +------------------------+
                                        |    Inventory Service   |
                                        +-----------+------------+
                                                    |
                                           Idempotency via Caffeine
                                                    |
                                              Inventory Updated




---

## ⚙️ Technologies Used

- **Java 21** + **Spring Boot**
- **RabbitMQ** (message broker)
- **Caffeine** (in-memory idempotency cache)
- **MongoDB** (Inventory data persistence)
- **PostgreSQL / H2** (for Orders)
- **Docker** (for RabbitMQ)
- **Swagger/OpenAPI** (API documentation)

---

## 🧠 Why No Circuit Breakers?

This is a fully **event-driven system**, meaning:

- Services **do not make synchronous calls** to each other.
- **RabbitMQ decouples communication**, so if Inventory Service is down, messages are safely queued.
- Retry and DLQ policies ensure eventual processing without needing a circuit breaker.
- **No cascading failures** because service failures don’t propagate upstream.

Circuit breakers are typically used to protect synchronous service calls (e.g., REST calls). In this architecture, they are unnecessary.

---

## 🧩 Idempotency with Caffeine Cache

To prevent duplicate processing of the same order:

```java
Cache<String, Boolean> processedOrders = Caffeine.newBuilder()
    .expireAfterWrite(5, TimeUnit.MINUTES)
    .maximumSize(10_000)
    .build();

if (processedOrders.getIfPresent(orderId) != null) {
    // Skip already processed order
    return;
}

processedOrders.put(orderId, true);
// Continue with inventory update

This prevents reprocessing due to retry logic, duplicate messages, or client resubmissions.


🧪 Simulating Load & Failure
You can simulate high-load and failure scenarios:

🔁 Bulk Order Posting
Send 100+ POST requests rapidly to OrderService:
for i in {1..100}; do
  curl -X POST http://localhost:8080/orders -H "Content-Type: application/json" \
  -d '{"orderId": "order-'$i'", "type": "IN_STORE", "storeId": "store-1"}'
done
for i in {1..100}; do
  curl -X POST http://localhost:8080/orders -H "Content-Type: application/json" \
  -d '{"orderId": "order-'$i'", "type": "IN_STORE", "storeId": "store-1"}'
done


