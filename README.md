# From Monolith to Microservices: Building a Lean, Observable MERN E-Commerce Stack

## Why This Project Exists

Every backend developer hits the same wall eventually. Tutorials walk you through building a REST API or a React app. What they skip is what happens when a real product gets too big for one server, one team, or one deploy pipeline.

Microservices is the industry's answer to that problem. But most real microservices codebases are enormous — thousands of files, dozens of services, years of accumulated infrastructure decisions. Not a great place to learn the fundamentals.

This project — a lean MERN microservices e-commerce stack — was built to solve exactly that. Small enough to read in an afternoon. Architecturally honest enough to demonstrate the patterns distributed systems teams use every day: an API gateway, event-driven messaging via RabbitMQ, Prometheus observability, and Kubernetes orchestration. It is a **structured learning artifact**, not a toy.

---

![](https://github.com/Pankaj-Leo/MERN_EComm_Distributed_system/blob/main/Info.png)
---
## What the Project Is

At its surface, this is a shopping application. You can sign up, log in, browse products, add items to a cart, and check out. The implementation is the point.

Instead of a single Node.js server handling everything, the application is split into **six independent services**, each running in its own process and container:

| Service      | What it does |
|--------------|-------------|
| **Auth**     | User registration, login, JWT token issuance |
| **Catalog**  | Product data management, image serving |
| **Cart**     | Session-based cart, checkout event publishing |
| **Order**    | Async order persistence, business metrics |
| **Shipping** | Async shipping record creation |
| **Gateway**  | Single entry point, traffic routing, HTTP metrics |

The React frontend talks only to the gateway. The gateway routes to the right service. Async communication between services flows through RabbitMQ. The entire system is observed via Prometheus and Grafana.

This mirrors the architecture of a mid-sized engineering organization — not Netflix-scale complexity, but far beyond a Hello World monolith.

---

## How the Project Was Built

### The API Gateway Pattern

The gateway is the first architectural decision worth understanding, and one of the most practically important.

Without a gateway, the React frontend would need to track the hostname and port of every service. That breaks the moment services move, scale, or split. With a gateway, the browser knows exactly one URL and everything else is hidden.

The gateway in this project is an Express application using `http-proxy-middleware` to forward requests:

```
/auth/*     → auth service (port 3001)
/catalog/*  → catalog service (port 3002)
/cart/*     → cart service (port 3003)
/order/*    → order service (port 3004)
/shipping/* → shipping service (port 3005)
/uploads/*  → catalog service (static product images)
```

Path rewriting ensures each service sees a clean path. A request for `/auth/login` arrives at the auth service as simply `/login`. This keeps services independently deployable without coupling their internal route structure to gateway conventions.

The gateway also handles CORS and tracks a single `http_requests_total` Prometheus counter — labeled by method, route, status code, and target service — giving you one metric that answers most operational traffic questions across the whole system.

### Authentication: JWT + bcrypt

The auth service implements two security fundamentals every user-facing application needs: password storage and session management.

Passwords are hashed with `bcryptjs` at 10 salt rounds before MongoDB ever sees them. Plain-text passwords are never stored. On login, `bcrypt.compare()` verifies the submitted password against the stored hash — without reversing it, which is the core property of bcrypt's design.

On successful login, the service issues a **JSON Web Token** signed with a secret key, carrying the user's ID and email, expiring after one hour. The frontend stores this in `localStorage` and sends it as a `Bearer` header on requests.

One deliberate gap: the cart and order services do not validate JWTs. This is intentional — it highlights exactly where auth middleware would be injected and makes it a clean, well-scoped next exercise.

### The Catalog and Static Files

The catalog service stores product documents in MongoDB, exposes a seed endpoint, and serves static product images from a local `uploads/` directory.

The seed endpoint is worth noting. Rather than a separate migration script, a `POST /seed` route checks the collection and inserts 8 products if it is empty. One HTTP call gets the service into a usable state. This is a practical pattern for demo and development environments that avoids the overhead of a separate CLI migration tool.

Product images are served by the catalog service itself via `express.static`. Unconventional for production (CDNs exist for this), but it demonstrates how static assets can be co-located with the service that owns them — and how the gateway's path rewriting makes image URLs transparent to the browser.

### The Cart and Event-Driven Checkout

The cart service contains the project's most instructive design decision: **the cart is in-memory**. Restart the service and cart contents vanish. This is documented in the codebase, not buried or hidden.

The reason is deliberate: in-memory storage keeps the implementation to a few dozen lines, so learners can focus on what actually matters — what happens at checkout.

When a user checks out, the cart service does not call the order service directly. It:

1. Serializes the cart as a JSON message
2. Publishes that message to a RabbitMQ queue named `orders`
3. Returns HTTP 200 to the browser immediately

The browser gets a fast response. Heavy downstream work happens asynchronously. This is the essence of event-driven architecture — the producer (cart) announces something happened and moves on. It has no knowledge of, or dependency on, the consumers.

### RabbitMQ: Multiple Consumers, One Queue

Both the order service and shipping service are consumers on the same `orders` queue. When a checkout message arrives, both receive it and process it independently.

The **order service** calculates the order total, saves an order document to MongoDB, and tracks three Prometheus metrics: an `orders_total` counter, an `orders_revenue_total` gauge, and an `order_processing_duration_seconds` histogram.

The **shipping service** creates a shipping record in MongoDB with status "processing" and increments an `orders_processed_total` counter.

Both services implement retry logic: if RabbitMQ is unavailable at startup, they retry up to 10 times at 5-second intervals. This is mandatory in containerized environments where startup order is not deterministic.

The multi-consumer pattern demonstrates something important at scale: one event can trigger completely different workflows — inventory, billing, analytics, notifications — with no direct coupling between any of them.

### Observability from Day One

Every service exposes a `/metrics` endpoint in Prometheus exposition format. Prometheus scrapes every endpoint on a 5-second interval. Grafana visualizes the data via two pre-provisioned dashboards that require no manual setup.

This is not decorative. The observability-first approach changes how you reason about services in production. When something breaks, the first questions are always: what was the request rate, the error rate, the latency? Without metrics, those questions have no answers.

The `order_processing_duration_seconds` histogram lets you calculate real percentile latencies — the p95 and p99 that SLAs are measured against:

```promql
histogram_quantile(0.95, rate(order_processing_duration_seconds_bucket[5m]))
```

The RabbitMQ Prometheus exporter adds queue-depth metrics. If orders accumulate faster than the order service can consume them, queue depth rises — an early warning signal before users notice delays.

### Docker Compose: One Command, Full Stack

`docker compose up --build` starts all 13 components: MongoDB, RabbitMQ, the RabbitMQ exporter, all six services, the React client, Prometheus, and Grafana. Health checks are defined for every service, and `depends_on` conditions enforce startup sequencing.

Named volumes persist MongoDB data across restarts. Prometheus and Grafana configurations are mounted from the repository, so the observability stack comes fully wired.

### Kubernetes: Same Architecture, Cluster-Ready

The `k8s/` directory contains one deployment manifest per service and one service manifest for in-cluster DNS. A ConfigMap holds all shared environment variables.

Each deployment includes readiness and liveness probes pointing to `/health` — the same endpoint Docker Compose uses. Services receive traffic only once they are actually ready.

Service names are identical in both environments: `mongo`, `rabbitmq`, `auth`, `catalog`. Docker Compose uses its default bridge network; Kubernetes uses in-namespace DNS. Same architecture, different orchestrator.

---

## Why This Project Matters

### Microservices Are Standard Practice at Scale

Single-process backends work well for getting started and for many production use cases. But growing engineering organizations hit structural problems: too many people committing to one codebase, too many features competing for the same deploy pipeline, too many domains entangled in one data model.

Microservices address those problems. Understanding the gateway pattern, service decomposition, async messaging, and independent deployability is not optional knowledge for a modern backend engineer — it is baseline.

### Event-Driven Architecture Transfers Everywhere

RabbitMQ in this project is a stand-in for a pattern used across the industry: Kafka at scale, SQS in AWS, Pub/Sub in GCP, Service Bus in Azure. The names change; the concept does not. Something happens — checkout — an event is published, and multiple downstream systems react independently.

In real e-commerce systems, a single "order placed" event triggers payment processing, inventory updates, email confirmation, fraud detection, and loyalty point accrual — all independently, all without direct coupling. Learning to design, produce, and consume events correctly is a durable, transferable skill.

### Observability Is Not a Feature — It Is an Operational Requirement

Microservices are harder to debug than monoliths because a single user request may touch four services. Without metrics and tracing, a failure in the order service is invisible until users report it. With metrics, you see the failure in real time on a Grafana dashboard — before the first support ticket.

The Prometheus and Grafana setup in this project demonstrates what observability-by-default looks like in practice. It is not a layer bolted on at the end; it is designed in from the first service.

### Infrastructure as Code Is a Core Competency

The `docker-compose.yml` and Kubernetes manifests in this project are not afterthoughts. They are first-class artifacts that encode exactly how the system runs. Any engineer can read the files, understand the topology, spin up the full stack locally, and make changes with confidence.

This is the infrastructure-as-code mindset. Small projects are the right place to build it — it transfers directly to Docker Compose, Kubernetes, Terraform, and every other tool in the ecosystem.

### Trade-off Literacy Is What Separates Architects from Implementers

The in-memory cart, shared MongoDB instance, missing JWT validation on protected routes, and single-replica Kubernetes deployments are all intentional. Each is a documented simplification with a clear production alternative.

Understanding not just what a system does but *why* specific choices were made — and what it would cost to change them — is what separates developers who write features from engineers who design systems. This codebase gives you the vocabulary for that conversation.

---
