# MERN Microservices E-Commerce — Lean Reference Stack

A **production-shaped, educational microservices platform** built with MongoDB, Express, React, and Node.js. Features a reverse-proxy API gateway, asynchronous order processing via RabbitMQ, full Prometheus + Grafana observability, Docker Compose local stack, and Kubernetes manifests for cluster deployment.

> **Goal:** Give developers a clear, runnable bridge from monolith tutorials to real distributed-systems concepts — without the noise of a large codebase.

---

![](https://github.com/Pankaj-Leo/MERN_EComm_Distributed_system/blob/main/Info.png)
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


```

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
## Author

**Pankaj Somkuwar** - AI Engineer / AI Product Manager / AI Solutions Architect

- LinkedIn: [Pankaj Somkuwar](https://www.linkedin.com/in/pankaj-somkuwar/)
- GitHub: [@Pankaj-Leo](https://github.com/Pankaj-Leo)
- Website: [Pankaj Somkuwar](https://www.pankajsomkuwarai.com)
- Email: [pankaj.som1610@gmail.com](mailto:pankaj.som1610@gmail.com)

