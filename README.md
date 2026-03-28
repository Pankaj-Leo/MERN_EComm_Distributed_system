# MERN Microservices E-Commerce — Lean Reference Stack

A **production-shaped, educational microservices platform** built with MongoDB, Express, React, and Node.js. Features a reverse-proxy API gateway, asynchronous order processing via RabbitMQ, full Prometheus + Grafana observability, Docker Compose local stack, and Kubernetes manifests for cluster deployment.

> **Goal:** Give developers a clear, runnable bridge from monolith tutorials to real distributed-systems concepts — without the noise of a large codebase.

---

## Table of Contents

- [Architecture](#architecture)
- [Services Overview](#services-overview)
- [Tech Stack](#tech-stack)
- [Repository Layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [Quick Start — Docker Compose](#quick-start--docker-compose)
- [Kubernetes Deployment](#kubernetes-deployment)
- [API Reference](#api-reference)
- [Monitoring & Observability](#monitoring--observability)
- [Configuration](#configuration)
- [Design Decisions & Trade-offs](#design-decisions--trade-offs)
- [Extending the Project](#extending-the-project)
- [Troubleshooting](#troubleshooting)

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│               Browser (React + Vite)                │
│                http://localhost:5173                 │
└────────────────────────┬────────────────────────────┘
                         │  HTTP
                         ▼
┌─────────────────────────────────────────────────────┐
│              API Gateway  :8080                      │
│   CORS · HTTP metrics · http-proxy-middleware        │
└──┬──────┬───────┬────────┬────────────┬─────────────┘
   │      │       │        │            │
   ▼      ▼       ▼        ▼            ▼
 Auth  Catalog  Cart    Order       Shipping
 :3001  :3002   :3003   :3004        :3005
   │      │       │        ▲            ▲
   └──────┴───────┘        │            │
          │             RabbitMQ ───────┘
          ▼            ("orders" queue)
       MongoDB
    (shared instance)

┌─────────────────────────────────────────────────────┐
│              Observability Layer                     │
│  Prometheus :9090  ←  /metrics on every service     │
│  Grafana    :3000  ←  pre-provisioned dashboards    │
│  RabbitMQ UI :15672                                 │
└─────────────────────────────────────────────────────┘
```

### Checkout Data Flow

```
1.  User clicks "Checkout" in React UI
2.  POST /cart/checkout → Gateway → Cart Service
3.  Cart serializes order JSON → publishes to RabbitMQ "orders" queue
4.  HTTP 200 returned immediately to browser
5a. Order Service   → reads message → saves order to MongoDB → updates Prometheus counters
5b. Shipping Service → reads same message → saves shipping record → increments shipment metric
```

---

## Services Overview

| Service      | Port  | Responsibility                                          | Key Tech                                      |
|--------------|-------|---------------------------------------------------------|-----------------------------------------------|
| **client**   | 5173  | React SPA — auth, catalog, cart, checkout UI            | React 18, Vite 6                              |
| **gateway**  | 8080  | Single HTTP entry point, reverse proxy, CORS, metrics   | Express, http-proxy-middleware                |
| **auth**     | 3001  | User signup/login, bcrypt password hashing, JWT issuance| Express, Mongoose, bcryptjs, jsonwebtoken     |
| **catalog**  | 3002  | Product listing, seed endpoint, static image hosting    | Express, Mongoose                             |
| **cart**     | 3003  | In-memory cart, publishes checkout events to RabbitMQ   | Express, amqplib                              |
| **order**    | 3004  | Consumes order events, persists to MongoDB, rich metrics | Express, amqplib, prom-client                |
| **shipping** | 3005  | Consumes order events, writes shipping records          | Express, amqplib, prom-client                 |
| **mongo**    | 27017 | Shared MongoDB instance                                 | MongoDB 7                                     |
| **rabbitmq** | 5672 / 15672 | Message broker + management UI                   | RabbitMQ 3                                    |
| **prometheus**| 9090 | Metrics collection                                     | Prometheus 2.55                               |
| **grafana**  | 3000  | Dashboards and visualization                            | Grafana 11                                    |

---

## Tech Stack

| Layer            | Technology                    | Version    |
|------------------|-------------------------------|------------|
| Frontend         | React, Vite                   | 18, 6.4    |
| API Gateway      | Express, http-proxy-middleware| 4.19, 3.0  |
| Backend Services | Node.js, Express              | 20+, 4.19  |
| Database ORM     | Mongoose                      | 8.19       |
| Database         | MongoDB                       | 7          |
| Message Broker   | RabbitMQ, amqplib             | 3, 0.10    |
| Auth             | bcryptjs, jsonwebtoken        | 2.4, 9.0   |
| Metrics Client   | prom-client                   | 15.1       |
| Monitoring       | Prometheus, Grafana           | 2.55, 11.2 |
| Containers       | Docker, Docker Compose        | —          |
| Orchestration    | Kubernetes                    | —          |

---

## Repository Layout

```
mern-microservices-lean-rebuilt-main/
├── client/                       # React + Vite frontend
│   ├── src/
│   │   ├── main.jsx              # App root — auth, catalog, cart views
│   │   └── components/
│   │       └── ProductGrid.jsx   # Product display grid
│   ├── Dockerfile                # Multi-stage build (build → serve)
│   └── vite.config.js
│
├── services/
│   ├── gateway/                  # API Gateway (CommonJS)
│   │   ├── index.js              # Proxy routes, health, metrics
│   │   └── metrics.js            # HTTP request counter middleware
│   ├── auth/                     # Authentication (ESM)
│   │   └── index.js              # /signup, /login, JWT
│   ├── catalog/                  # Product catalog (ESM)
│   │   ├── index.js              # /products, /seed, /uploads static
│   │   ├── models/Product.js
│   │   └── uploads/              # 8 demo product JPGs
│   ├── cart/                     # Shopping cart (ESM)
│   │   └── index.js              # In-memory cart, RabbitMQ publish
│   ├── order/                    # Order processor (ESM)
│   │   ├── index.js              # RabbitMQ consumer, MongoDB persistence
│   │   └── metrics.js            # Counter, revenue gauge, histogram
│   └── shipping/                 # Shipping processor (ESM)
│       ├── index.js              # RabbitMQ consumer, shipping records
│       └── metrics.js            # Shipments counter
│
├── k8s/                          # Kubernetes manifests
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── *-deployment.yaml         # One per service + infra
│   └── README.txt
│
├── monitoring/
│   ├── prometheus/
│   │   └── prometheus.yml        # Scrape config (5s interval)
│   └── grafana/
│       ├── provisioning/         # Auto-provision datasource + dashboards
│       └── dashboards/           # microservices.json, order-shipping.json
│
├── docker-compose.yml            # Full 13-service local stack
├── .env.example                  # Environment variable template
└── patch.json                    # kubectl patch examples
```

---

## Prerequisites

- **Docker + Docker Compose** (recommended path — no local Node.js needed)
- **Node.js 20+** (only if running services outside Docker)
- **kubectl + a Kubernetes cluster** (for K8s deployment only)

---

## Quick Start — Docker Compose

### 1. Clone and configure

```bash
git clone <repo-url>
cd mern-microservices-lean-rebuilt-main
cp .env.example .env
# Set a strong JWT_SECRET in .env before proceeding
```

### 2. Start the full stack

```bash
docker compose up --build
```

All 13 services start with health checks and dependency ordering. First boot pulls images and compiles the frontend — allow 2–3 minutes.

### 3. Seed the product catalog

```bash
curl -X POST http://localhost:8080/catalog/seed
```

Inserts 8 demo products (desk lamp, office chair, sofa, etc.) with real images. Safe to run multiple times — the endpoint is idempotent.

### 4. Access the stack

| URL                        | Service              |
|----------------------------|----------------------|
| http://localhost:5173      | React frontend       |
| http://localhost:8080      | API Gateway          |
| http://localhost:9090      | Prometheus           |
| http://localhost:3000      | Grafana              |
| http://localhost:15672     | RabbitMQ Management UI (guest / guest) |

### 5. Try the full flow

1. Open http://localhost:5173
2. Sign up with an email and password
3. Log in — the product grid loads
4. Add items to cart
5. Click Checkout — the order is published to RabbitMQ
6. Watch `orders_total` increment at http://localhost:9090

### Stopping the stack

```bash
docker compose down          # Stop containers, keep volumes (MongoDB data survives)
docker compose down -v       # Stop and wipe all volumes
```

---

## Kubernetes Deployment

### Deploy everything

```bash
kubectl apply -f k8s/
```

Creates the `mern-ecommerce` namespace and deploys all services and infrastructure.

### Port-forward for local access

```bash
kubectl port-forward svc/gateway    -n mern-ecommerce 8080:8080
kubectl port-forward svc/client     -n mern-ecommerce 5173:5173
kubectl port-forward svc/prometheus -n mern-ecommerce 9090:9090
kubectl port-forward svc/grafana    -n mern-ecommerce 3000:3000
```

### Check status

```bash
kubectl get pods -n mern-ecommerce
kubectl get svc  -n mern-ecommerce
```

### Update a service image

```bash
kubectl patch deployment auth -n mern-ecommerce --patch-file patch.json
```

Global environment variables live in `k8s/configmap.yaml`. Per-service overrides go in the relevant `*-deployment.yaml`.

---

## API Reference

All requests route through the **API Gateway at port 8080**.

### Auth

| Method | Path           | Body                                      | Response                        |
|--------|----------------|-------------------------------------------|---------------------------------|
| `POST` | `/auth/signup` | `{ "email": "...", "password": "..." }`   | `{ "message": "User registered" }` |
| `POST` | `/auth/login`  | `{ "email": "...", "password": "..." }`   | `{ "token": "<JWT>" }`          |

```bash
# Sign up
curl -X POST http://localhost:8080/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"secret"}'

# Log in
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"secret"}'
```

### Catalog

| Method | Path                   | Response                       |
|--------|------------------------|--------------------------------|
| `GET`  | `/catalog/products`    | Array of product documents     |
| `POST` | `/catalog/seed`        | Seeds 8 demo products (idempotent) |
| `GET`  | `/uploads/{filename}`  | Static product image           |

### Cart

| Method | Path               | Body                                 | Response                                 |
|--------|--------------------|--------------------------------------|------------------------------------------|
| `POST` | `/cart/add`        | `{ "productId": "...", "qty": 1 }`   | Updated cart array                       |
| `GET`  | `/cart/items`      | —                                    | Current cart array                       |
| `POST` | `/cart/checkout`   | —                                    | `{ "ok": true, "message": "Order placed!" }` |

### Orders

| Method | Path           | Body                   | Response                       |
|--------|----------------|------------------------|--------------------------------|
| `POST` | `/order/create`| `{ "items": [...] }`   | `{ "ok": true, "order": {...} }` |

### Health & Metrics

Every service exposes:
- `GET /health` → `"OK"` (used by Docker and Kubernetes probes)
- `GET /metrics` → Prometheus text format

---

## Monitoring & Observability

### Prometheus Metrics

| Service           | Metric                               | Type      | Description                               |
|-------------------|--------------------------------------|-----------|-------------------------------------------|
| All services      | `http_requests_total`                | Counter   | Requests by method / route / status       |
| Order             | `orders_total`                       | Counter   | Total orders created                      |
| Order             | `orders_failed_total`                | Counter   | Failed order processing attempts          |
| Order             | `orders_revenue_total`               | Gauge     | Cumulative revenue (USD)                  |
| Order             | `order_processing_duration_seconds`  | Histogram | Processing time distribution (p50/p95/p99)|
| Shipping          | `orders_processed_total`             | Counter   | Shipments processed                       |
| RabbitMQ Exporter | queue metrics                        | Various   | Queue depth, consumer count, etc.         |

### Grafana Dashboards

Two dashboards are auto-provisioned at startup — no manual configuration required:

- **Microservices** — HTTP request rates, error rates, latency by service
- **Order & Shipping** — Business metrics: order volume, revenue, shipping throughput

Access at http://localhost:3000. Credentials are set in `docker-compose.yml`.

### Useful PromQL Queries

```promql
# Request rate across all services (5-minute window)
rate(http_requests_total[5m])

# Order processing latency at p95
histogram_quantile(0.95, rate(order_processing_duration_seconds_bucket[5m]))

# Total orders ever placed
orders_total

# Cumulative revenue
orders_revenue_total
```

---

## Configuration

### Environment Variables

| Variable        | Default                          | Description                                     |
|-----------------|----------------------------------|-------------------------------------------------|
| `JWT_SECRET`    | `dev_change_me`                  | JWT signing secret — **change before any deployment** |
| `MONGO_URL`     | `mongodb://mongo:27017/appdb`    | MongoDB connection string                       |
| `RABBIT_URL`    | `amqp://rabbitmq:5672`           | RabbitMQ connection string                      |
| `AMQP_URL`      | `amqp://rabbitmq:5672`           | Alias used by order and shipping services       |
| `VITE_API_BASE` | `http://localhost:8080`          | Gateway URL consumed by the React frontend      |
| `GATEWAY_PORT`  | `8080`                           | Gateway listen port                             |
| `AUTH_PORT`     | `3001`                           | Auth service port                               |
| `CATALOG_PORT`  | `3002`                           | Catalog service port                            |
| `CART_PORT`     | `3003`                           | Cart service port                               |
| `ORDER_PORT`    | `3004`                           | Order service port                              |

Copy `.env.example` → `.env` and adjust values before running Docker Compose.

---

## Design Decisions & Trade-offs

These are intentional simplifications, not oversights. Each has a documented production path.

| Decision                                        | Rationale                                      | Production Alternative                               |
|-------------------------------------------------|------------------------------------------------|------------------------------------------------------|
| **In-memory cart**                              | Keeps cart code to ~50 lines; checkout is the focus | Redis or MongoDB-backed cart per user session   |
| **Shared MongoDB instance**                     | Reduces infra complexity for learning          | Separate DB per service (true data isolation)        |
| **Order + Shipping on same queue**              | Demonstrates multi-consumer fan-out            | Topic exchange with routing keys + idempotency keys  |
| **No JWT validation on cart/order routes**      | Avoids boilerplate that obscures the architecture | Auth middleware injected per-service or at gateway |
| **Single replica in Kubernetes**                | Clarity — one pod is easier to follow          | HPA + PodDisruptionBudget for real HA                |
| **Secrets in ConfigMap**                        | Dev convenience                                | Kubernetes Secrets + external Vault integration      |
| **No request body validation**                  | Reduces setup noise                            | Zod or Joi schema middleware on every POST route     |

---

## Extending the Project

Ordered roughly by difficulty — each one closes a real production gap:

1. **Add JWT middleware** to cart and order routes — verify the `Authorization: Bearer` header before allowing checkout
2. **Make the cart persistent** — replace the in-memory array with a Redis-backed or MongoDB-backed user cart
3. **Add idempotency** to the order consumer — use a `messageId` field to prevent duplicate order inserts on retry
4. **Add a notifications service** — a third RabbitMQ consumer that sends email on order confirmation
5. **Introduce distributed tracing** — OpenTelemetry instrumentation routed to Jaeger
6. **Add Horizontal Pod Autoscaler** — scale the Order service based on RabbitMQ queue depth
7. **Add integration tests** — Testcontainers for real MongoDB and RabbitMQ in CI
8. **Implement API versioning** — prefix all routes with `/v1/`

---

## Troubleshooting

**Services not starting**
```bash
docker compose ps           # Identify unhealthy containers
docker compose logs auth    # View logs for a specific service
```

**RabbitMQ connection errors in Order / Shipping**
Order and shipping retry up to 10 times at 5-second intervals. If errors persist beyond 50 seconds, RabbitMQ itself failed to start — check `docker compose logs rabbitmq`.

**MongoDB connection refused**
`MONGO_URL` uses the service name `mongo` inside Docker networks. If running services locally (outside Docker), change it to `localhost:27017`.

**Client shows blank page or CORS errors**
Confirm `VITE_API_BASE` points to the gateway (`http://localhost:8080`). The gateway sets `Access-Control-Allow-Origin: *` — if CORS errors appear, the request is bypassing the gateway.

**Grafana shows no data**
Check http://localhost:9090/targets — all scrape targets should show state `UP`. If a target is `DOWN`, the service's `/metrics` endpoint is unreachable from the Prometheus container.

**Full reset**
```bash
docker compose down -v --remove-orphans
docker compose up --build
curl -X POST http://localhost:8080/catalog/seed
```

---

## License

Educational and portfolio reference implementation. Audit secrets, authentication coverage, and scaling configuration before any production use.
