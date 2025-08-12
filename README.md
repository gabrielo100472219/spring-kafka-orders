# spring-kafka-orders

## What you’ll build

A small e-commerce backend split into **three Spring Boot services**:

1. **Order Service (REST)**

   * Accepts orders, persists them in **MySQL**.
   * Publishes domain events to **Kafka**.
2. **Inventory Service (Kafka + REST)**

   * Listens to order events, reserves stock in **MySQL**.
   * Replies via Kafka with reservation results.
3. **Notification Service (Kafka)**

   * Listens to “order confirmed / failed” events.
   * Simulates sending emails/SMS (just logs).

Optional later: a tiny **Payment Service** that “authorizes” payments (random pass/fail) via Kafka.

---

## High-level flow (happy path)

1. `POST /api/orders` → Order Service validates & creates `PENDING` order → writes to MySQL **and** enqueues an **outbox** record.
2. An internal **Outbox Publisher** (in Order Service) publishes `order.created` to Kafka (then marks outbox row as sent).
3. Inventory Service consumes `order.created`, checks/reserves stock in MySQL, emits either:

   * `inventory.reserved` (ok) or
   * `inventory.rejected` (insufficient stock).
4. Order Service consumes the inventory result and updates the order:

   * `CONFIRMED` on success (and emits `order.confirmed`)
   * `FAILED` on rejection (and emits `order.failed`).
5. Notification Service consumes `order.confirmed` / `order.failed` and logs a “sent email”.

This teaches **event-driven coordination**, **idempotency**, and **eventual consistency** without going overboard.

---

## Tech & dependencies

* Spring Boot (Web, Validation, Spring Data JPA, Spring for Kafka, Actuator)
* MySQL 8
* Kafka (dev: Docker Compose with Bitnami/Confluent)
* Flyway or Liquibase for schema migrations
* Testcontainers (MySQL + Kafka) for integration tests
* Lombok, MapStruct (optional)

---

## Domain model (minimum)

### Order Service (MySQL)

* `orders`

  * `id` (PK, UUID)
  * `customer_email`
  * `status` (PENDING, CONFIRMED, FAILED, CANCELED)
  * `total_amount`
  * `created_at`, `updated_at`
* `order_items`

  * `id` (PK)
  * `order_id` (FK)
  * `sku`
  * `unit_price`
  * `quantity`
* `outbox`

  * `id` (PK)
  * `aggregate_type` (e.g., ORDER)
  * `aggregate_id` (order id)
  * `type` (e.g., order.created)
  * `payload` (JSON)
  * `headers` (JSON)
  * `status` (NEW, SENT, ERROR)
  * `created_at`, `sent_at`

### Inventory Service (MySQL)

* `inventory`

  * `sku` (PK)
  * `available_qty`
  * `reserved_qty`
* `reservation_log` (for idempotency & audit)

  * `event_id` (PK from Kafka record key)
  * `order_id`
  * `sku`
  * `delta_reserved`
  * `status` (RESERVED/REJECTED)
  * `created_at`

---

## Kafka topics (start with 1–3 partitions)

* `order.created` (key: `orderId`)
* `inventory.reserved` (key: `orderId`)
* `inventory.rejected` (key: `orderId`)
* `order.confirmed` (key: `orderId`)
* `order.failed` (key: `orderId`)

Add **DLQs**:

* `order.created.dlq`, `inventory.reserved.dlq`, …

---

## REST endpoints (Order Service)

* `POST /api/orders`

  * Request: `{ customerEmail, items: [{ sku, quantity, unitPrice }] }`
  * Response: `{ orderId, status: "PENDING" }`
  * Headers: support `Idempotency-Key` to avoid duplicate orders.
* `GET /api/orders/{id}` → order details + status
* `POST /api/orders/{id}/cancel` (emits `order.canceled` later if you add refunds)

---

## Key patterns to practice

1. **Transactional Outbox (no dual writes!)**

   * In the same DB transaction that creates the order, write an `outbox` row with the event JSON.
   * A scheduled publisher (or Spring @TransactionalEventListener + background) reads NEW rows, publishes to Kafka, marks SENT.
   * Retries & error handling put rows into `ERROR` with backoff.

2. **Idempotent consumers**

   * Use the Kafka record key (`orderId`) and an `event_id` (e.g., `orderId#version`) in consumer tables to ensure **exactly-once effects** at your app layer.

3. **Schema versioning**

   * Add `type` and maybe `version` to event payloads. Keep payloads compact and serializable as JSON (Jackson). Later, try Avro/Confluent Schema Registry.

4. **Error handling & DLQ**

   * On deserialization/validation failures, produce to `.dlq` with original headers + error reason. Expose an actuator metric.

5. **Consumer concurrency**

   * Start with `max.poll.records=1` and then tune. Use `@KafkaListener(concurrency = "3")` after you’re comfortable.

---

## Milestones (suggested order)

1. **Skeleton & Compose**

   * Three Spring Boot apps in a single repo (or multi-repo).
   * `docker-compose.yml` with MySQL, Kafka (and ZooKeeper if needed), and app containers.
   * Flyway migrations for both databases.

2. **Order creation + Outbox publish**

   * `POST /api/orders` saves order + outbox (NEW).
   * Background outbox publisher emits `order.created`.

3. **Inventory reservation**

   * Inventory Service consumes `order.created`.
   * Checks `inventory.available_qty >= sum(orderItems)`.
   * On success: update `reserved_qty`, emit `inventory.reserved`.
   * On fail: emit `inventory.rejected`.

4. **Order state transitions + notifications**

   * Order Service consumes inventory results and updates status (CONFIRMED/FAILED) + emits `order.confirmed`/`order.failed`.
   * Notification Service logs “Email to X: your order is …”.

5. **Idempotency & DLQs**

   * Add `Idempotency-Key` on create order; store key hash in `orders` to reject duplicates gracefully.
   * Implement DLQ production on consumer errors.

6. **Tests**

   * **Unit:** services, mappers, validators.
   * **Integration:** Testcontainers MySQL & Kafka; end-to-end flow: create order → inventory reserve → order confirmed → notification received.

7. **Observability**

   * Actuator `/health`, `/info`, `/metrics`.
   * Basic logs with correlation id (propagate from HTTP → Kafka headers).

---

## Example event payloads (JSON)

`order.created`

```json
{
  "eventType": "order.created",
  "version": 1,
  "orderId": "f7a1...",
  "customerEmail": "alice@example.com",
  "items": [
    {"sku":"SKU-123","quantity":2,"unitPrice":1299}
  ],
  "totalAmount": 2598,
  "createdAt": "2025-08-12T10:23:01Z"
}
```

`inventory.reserved`

```json
{
  "eventType": "inventory.reserved",
  "version": 1,
  "orderId": "f7a1...",
  "reservations": [
    {"sku":"SKU-123","quantity":2}
  ]
}
```

---

## Suggested package structure (each service)

```
com.example.orders
  ├─ api (controllers, DTOs, validators)
  ├─ domain (entities, aggregates)
  ├─ infra
  │   ├─ jpa (repositories)
  │   ├─ kafka (producers, consumers, configs)
  │   └─ outbox (publisher, repo)
  └─ app (use-cases/services)
```

---

## Docker Compose sketch (services + infra)

* **MySQL**: expose 3306, init scripts for both DBs (or two schemas)
* **Kafka**: Bitnami images are straightforward (`bitnami/zookeeper`, `bitnami/kafka`)
* **Apps**: build JARs and run with env vars for DB & Kafka brokers

(*If you want, I can generate a ready-to-run `docker-compose.yml` and the Spring `application.yml` templates next.*)

---

## Stretch goals (when you’re comfy)

* Replace outbox publisher with **Debezium** CDC connector.
* Add **Payment Service** with a **saga**: Order → Payment → Inventory → Order finalization.
* Use **Avro + Schema Registry** and practice schema evolution.
* Add **Keycloak** (or Spring Security) for customer auth.
* Add **Grafana + Prometheus** to watch consumer lag & business KPIs.

---

If you like this plan, I can:

* draft the **DB schemas (Flyway migrations)**,
* write **topic configs** and **Kafka properties**,
* and give you a **minimal code skeleton** for the Order Service (controller, entities, outbox publisher) to get you coding right away.
