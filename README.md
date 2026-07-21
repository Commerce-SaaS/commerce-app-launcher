# Commerce App Launcher

A microservices-based commerce/restaurant-POS platform. This repository is the orchestration layer: it holds the root `docker-compose.yml`, shared environment configuration, an observability stack, and each backend service as a Git submodule. It contains no application code of its own.

All eight services are NestJS applications written in TypeScript, communicating over RabbitMQ (RPC request/response and topic-exchange events). PostgreSQL (via TypeORM) and Redis are shared infrastructure; several services also integrate with Stripe and AWS S3.

**This README, and the per-service READMEs it links to, document only what was verified directly in the source code of this repository as of 2026-07-16. Anything the audit could not confirm is marked "TODO: verify" — including in the linked service READMEs. Do not treat unmarked statements below as more certain than the underlying service README.**

---

## Services

| Service | Directory | Role | Real port | Public HTTP API |
|---|---|---|---|---|
| Client Gateway | [`client-gateway/`](client-gateway/README.md) | Single HTTP entry point for clients (SaaS dashboard + storefront); auths requests, proxies to backend services over RabbitMQ, terminates Stripe webhooks | 4000 | Yes (only service with HTTP routes) |
| Auth MS | [`auth-ms/`](auth-ms/README.md) | Authentication/session/profile management for SaaS staff users and organization customers (JWT, Redis sessions, Google OAuth) | 4002 (RMQ only; no HTTP listener is bound — see note below) | No |
| Product MS | [`product-ms/`](product-ms/README.md) | Product catalog: products, categories, tags, ingredients, extras | 4001 (RMQ only; no HTTP listener) — also runs a real Prometheus `/metrics` server on 9100 | No |
| Organization MS | [`organization-ms/`](organization-ms/README.md) | Organizations, organization domains, user↔organization memberships/roles | 4003 (RMQ only; no HTTP listener) | No |
| Orders MS | [`orders-ms/`](orders-ms/README.md) | Orders (dine-in/POS/scheduled), order items, tables, sectors, cash sessions, sales analytics for a restaurant/POS platform | 4007 (RMQ only; no HTTP listener) | No |
| Payments MS | [`payments-ms/`](payments-ms/README.md) | Payments, payment methods, subscriptions, Stripe (Checkout, Subscriptions, Connect Express) | 4006 (RMQ only; no HTTP listener) | No |
| Media MS | [`media-ms/`](media-ms/README.md) | File upload/delete backed by AWS S3 | No HTTP/TCP port is actually bound (see note below) | No |
| Notifications MS | [`notifications-ms/`](notifications-ms/README.md) | Consumes events and sends transactional email (password reset, email verification, email-changed) via SMTP + Handlebars templates | No HTTP/TCP port is actually bound (see note below) | No |

**Important, verified across multiple services:** every backend microservice (everything except `client-gateway`) is bootstrapped with `NestFactory.createMicroservice(..., { transport: Transport.RMQ })` and does **not** call `app.listen(port)` for HTTP. The `PORT` env var each service reads is only used in a startup log line in most of them. The "real port" numbers in the table above are what `docker-compose.yml`, each service's Dockerfile, and `.env.example` agree on for container port mapping — they do not necessarily correspond to an actual listening TCP port inside the container. Several services have inconsistent `EXPOSE`/`.env.example`/compose port values; see each service's README for the specific discrepancy. `client-gateway` is the only service confirmed to open a real HTTP listener.

Only `client-gateway` exposes an HTTP API. All inter-service communication (both from the gateway and between backend services) happens exclusively over RabbitMQ — no service was found making direct HTTP or gRPC calls to another.

---

## Architecture

```
                         ┌────────────┐
   Client (web/app) ───► │   client-  │  HTTP :4000  (only HTTP entry point)
                         │  gateway   │
                         └─────┬──────┘
                               │  RabbitMQ RPC (per-service queues)
       ┌───────────┬───────────┼───────────┬───────────┬───────────┐
       ▼           ▼           ▼           ▼           ▼           ▼
   auth-ms    product-ms  organization- orders-ms  payments-ms  media-ms
  (auth_queue) (products_  ms (organi-  (orders_   (payments_   (media_
               queue)      zation_      queue)      queue)      queue)
                            queue)
       │           │           │           │           │
       │           │           │           │           │
       └── PostgreSQL (one DB per service) ─┴───────────┘
                               │
                    Redis (sessions / cache / idempotency)
                               │
             ┌─────────────────────────────────┐
             │   RabbitMQ topic exchange        │
             │        "app.events"              │
             └─────┬───────────────┬─────────────┘
                    │               │
             notifications-ms   (cross-service event fan-out,
             (SMTP, no DB)       see "Event Flows" below)
```

`media-ms` and `notifications-ms` have no database. `notifications-ms` also has no outbound queue of its own (pure consumer). Diagram reflects only connections found in code — see per-service READMEs for exact queue names and event lists.

### Verified event flows (topic exchange `app.events`)

These are the cross-service RabbitMQ events actually found in the code (not inferred):

- **auth-ms → notifications-ms**: `verify_email.saas`, `verify_email.customer`, `forgot_password.saas`, `forgot_password.customer`, `saas_mailer_email_changed`, `customer_mailer_email_changed` — all consumed by `notifications-ms` on its events queue and rendered as transactional emails.
- **auth-ms → orders-ms, payments-ms, organization-ms**: `customer.anonymized` (fan-out to all three) — each service nulls/soft-deletes the corresponding customer's PII/records.
- **auth-ms → (authz consumer)**: `userOrganization.user_authz_refresh` — consumed by `organization-ms`, which rebuilds a Redis-cached list of a user's organizations/roles.
- **orders-ms → payments-ms**: `order.cancelled` — cancels active payments for the order. (Consumer confirmed in `payments-ms`.)
- **payments-ms → orders-ms**: `payment.status` — orders-ms updates order state on payment success/failure.
- **payments-ms → organization-ms**: `organization.update_event`, `organization.clear_stripe_account`/`organization.stripe.clear` — clears a Stripe Connect account reference on the organization.
- **orders-ms → organization-ms**: RPC call `organization.find_one` (request/response, not an event) — orders-ms reads an organization's scheduling config (`orderSchedulingIntervalMinutes`, `maxDishesPerSlot`, `openingHours`) when validating scheduled orders.
- **client-gateway → all backend services**: RPC request/response over each service's dedicated queue (`auth_queue`, `organization_queue`, `products_queue`, `orders_queue`, `payments_queue`, `media_queue`) for every proxied HTTP request. The gateway also has dedicated outbound clients for two event-only queues (`RMQ_EVENTS_QUEUE_ORGANIZATION`, `RABBITMQ_QUEUE_EVENTS_PAYMENTS`).

**Stripe webhooks — TODO: verify.** `payments-ms` has no HTTP endpoint and no Stripe signature-verification code; it receives Stripe webhook data as a RabbitMQ event (`payment.webhook`). `client-gateway` does expose `POST /webhooks/platform` and `POST /webhooks/connect` with real Stripe signature verification, but the audit of `payments-ms` could not independently confirm that the gateway is the service that republishes the verified event onto `payment.webhook` — this wiring is plausible but unverified end-to-end.

---

## Stack

Verified per-service from each `package.json`:

- **Language**: TypeScript 5.7
- **Framework**: NestJS 11 (`@nestjs/core`, `@nestjs/common`, `@nestjs/microservices`)
- **Database**: PostgreSQL 16.2 (Docker image `postgres:16.2`) via TypeORM 0.3 — one database per service (auth-ms, product-ms, organization-ms, orders-ms, payments-ms have their own DB; media-ms and notifications-ms have none)
- **Messaging**: RabbitMQ (`rabbitmq:3-management` image) via `@nestjs/microservices` (`Transport.RMQ`), `amqplib`, `amqp-connection-manager`
- **Cache/session store**: Redis 7 (`redis:7` image) via `ioredis`
- **Env validation**: `zod` in every service (`src/config/envs.ts`) — each service throws at startup if required variables are missing
- **Payments**: `stripe` SDK (client-gateway and payments-ms)
- **File storage**: `@aws-sdk/client-s3` (media-ms)
- **Email**: `@nestjs-modules/mailer` + `nodemailer` + `handlebars` (notifications-ms)
- **Runtime**: Node 20 (Alpine) in every Dockerfile
- **Observability**: Prometheus (`prom/prometheus:v2.53.0`), Grafana (`grafana/grafana:11.1.0`), `redis_exporter` — see `observability/` and the docker-compose services below. Of the application services, only `product-ms` has real, code-level Prometheus instrumentation (`prom-client`, a `/metrics` HTTP server on port 9100).

### Prerequisites

- Docker and Docker Compose (root `docker-compose.yml` targets the Compose v2 `services:` schema)
- Node.js 20+ and npm, only needed for running a service outside Docker
- Git, with submodule support (`git submodule update --init --recursive`)

---

## Running the Full Stack

Commands verified against the root `docker-compose.yml` and `.env.example`.

```bash
git clone <repository-url>
cd commerce-app-launcher
git submodule update --init --recursive

cp .env.example .env
# then fill in real secrets — see "Environment Variables" below

docker compose up --build
```

This brings up, per `docker-compose.yml`: `redis`, `rabbitmq`, `client-gateway`, `products-ms` + `products-db`, `auth-ms` + `auth-db`, `organization-ms` + `organization-db`, `orders-ms` + `orders-db`, `payments-ms` + `payments-db`, `media-ms`, `notifications-ms`, plus an observability stack (`prometheus`, `redis-exporter`, `grafana`).

Each application service runs `npm run start:dev` inside its container with the service directory bind-mounted for live reload (per the `volumes:` entries in `docker-compose.yml`).

### Port mappings (from `docker-compose.yml`)

| Service | Host port |
|---|---|
| client-gateway | 4000 |
| products-ms | 4001 (+ metrics 9100 on `127.0.0.1` only) |
| auth-ms | 4002 |
| organization-ms | 4003 |
| media-ms | 4004 |
| notifications-ms | 4005 |
| payments-ms | 4006 |
| orders-ms | 4007 |
| RabbitMQ AMQP | 5672 |
| RabbitMQ management UI | 15672 |
| Redis | 6379 (`127.0.0.1` only) |
| products-db (Postgres) | 5432 (`127.0.0.1` only) |
| auth-db | 5433 |
| organization-db | 5434 |
| payments-db | 5435 |
| orders-db | 5436 |
| Prometheus UI | 9090 (`127.0.0.1` only) |
| Grafana UI | 3001 (`127.0.0.1` only) → container port 3000 |

As noted above, most of these "service" ports (4001–4007, excluding 4000) are container-port mappings from `docker-compose.yml`/`.env.example` that do **not** correspond to an HTTP port the Nest app actually binds — the services are RabbitMQ-only. Treat them as informational/reserved, not as working HTTP endpoints, unless a service's own README says otherwise.

### Running a single service outside Docker

```bash
docker compose up redis rabbitmq auth-db   # infra only, example for auth-ms
cd auth-ms
npm install
cp .env.example .env   # then fill in values; auth-ms's own .env.example is missing several vars its schema requires — see auth-ms/README.md
npm run start:dev
```

Repeat with the relevant `*-db` service(s) for other backend services. Each service's own README documents its exact required environment variables and known gaps in its `.env.example`.

### Stopping / cleaning up

```bash
docker compose down          # stop and remove containers
docker compose down -v       # also remove named volumes (deletes DB/Redis/Prometheus/Grafana data)
```

---

## Environment Variables

The root `.env.example` lists the variables `docker-compose.yml` substitutes into each service's container environment. **It is not exhaustive** — the per-service audits found required variables (per each service's own Zod schema) that are missing from both the root `.env.example` and that service's own `.env.example`. Each service's README documents its full, verified variable list and flags exactly which ones are missing from the example files. Do not assume the root `.env.example` alone is sufficient to boot every service; consult the linked service READMEs.

Root `.env.example` groups (see file for exact keys): `NODE_ENV`, RabbitMQ connection (`RABBITMQ_USER`/`PASS`/`HOST`/`PORT`/`URL`), Redis (`REDIS_HOST`/`PORT`; note **`REDIS_PASS` is not in the root `.env.example`** even though multiple services require it), frontend URL (`CLIENT_URL`), JWT secrets (`JWT_SECRET_ACCESS`/`REFRESH`/`RESET` — **`JWT_SECRET_VERIFY_EMAIL`, required by auth-ms, is missing**), Stripe (`STRIPE_SECRET`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_CONNECT_WEBHOOK_SECRET` — **`STRIPE_PRICE_ID_BASIC`/`STRIPE_PRICE_ID_PRO`, required by payments-ms, are missing**), Google OAuth (`GOOGLE_CLIENT_ID`), SMTP (`SMTP_HOST`/`PORT`/`USER`/`PASS`/`FROM` — **`LOGO_URL`, `APP_NAME`, `APP_URL`, required by notifications-ms, are missing**), AWS (`AWS_ACCESS_KEY_ID`/`SECRET_ACCESS_KEY`/`REGION`/`BUCKET`), database credentials and per-service DB names, and per-service RabbitMQ queue name variables.

---

## Repository Structure

```
commerce-app-launcher/
├── docker-compose.yml       # Orchestrates all services + infra + observability
├── .env.example              # Template for shared/root environment variables (incomplete — see above)
├── .gitmodules                # Each service below is a separate Git submodule
├── observability/             # Prometheus + Grafana provisioning config
├── client-gateway/            # HTTP API gateway — see client-gateway/README.md
├── auth-ms/                   # Authentication/session service — see auth-ms/README.md
├── product-ms/                # Product catalog service — see product-ms/README.md
├── organization-ms/           # Organization/membership service — see organization-ms/README.md
├── orders-ms/                 # Orders/POS service — see orders-ms/README.md
├── payments-ms/                # Payments/subscriptions/Stripe service — see payments-ms/README.md
├── media-ms/                  # S3 file storage service — see media-ms/README.md
└── notifications-ms/           # Transactional email service — see notifications-ms/README.md
```

Each service directory is a Git submodule (see `.gitmodules`) tracking its own repository on the `develop` branch.

---

## Per-Service Documentation

- [Client Gateway](client-gateway/README.md) — HTTP routes, guards, downstream RabbitMQ queue map, Stripe webhook handling
- [Auth MS](auth-ms/README.md) — all RabbitMQ message patterns, JWT/session model, outbound events
- [Product MS](product-ms/README.md) — catalog message patterns, data model, Prometheus metrics
- [Organization MS](organization-ms/README.md) — organization/membership message patterns, Redis cache scheme
- [Orders MS](orders-ms/README.md) — order/table/sector/cash-session message patterns, inter-service calls
- [Payments MS](payments-ms/README.md) — payment/subscription/Connect message patterns, what Stripe features are actually implemented
- [Media MS](media-ms/README.md) — S3 upload/delete message patterns
- [Notifications MS](notifications-ms/README.md) — consumed email events and templates

---

## Known Gaps Across the Platform

Patterns repeated across nearly every service's audit, worth tracking centrally:

- **No test files exist** in auth-ms, organization-ms, orders-ms (partial — a few `.spec.ts` exist), payments-ms (2 spec files), media-ms, notifications-ms, and product-ms, despite `test`/`test:e2e` scripts being configured in every `package.json`. `client-gateway` has real, extensive `.spec.ts` coverage but its `test:e2e` script is also broken (no `test/` directory).
- **Every backend service's `.env.example` is missing at least one variable its own Zod schema requires** (commonly `REDIS_PASS`) — copying `.env.example` as-is will fail startup validation in most services. See each service's README for the exact missing keys.
- **No database migrations exist anywhere** — every service relies on TypeORM `synchronize: true` in development; production migration strategy is not documented in any service's code.
- **Several services define RabbitMQ pattern constants or client tokens with no corresponding implementation** (dead code) — e.g. unimplemented `find_all`/`restore` patterns in organization-ms and product-ms's join-entity controllers, unused `RabbitMQModule`/`RMQ_SERVICE` clients in media-ms and product-ms. Full detail in each service's README.
- **Port/`.env.example`/Dockerfile inconsistencies** exist in payments-ms, media-ms, and orders-ms (the value in the Dockerfile's `EXPOSE`, `.env.example`'s `PORT`, and/or the Zod schema's default disagree). Since none of these services bind an actual HTTP port, the practical impact is limited to documentation confusion — see each README.
- **Refunds and PayPal are not implemented** in payments-ms despite related enum values/pattern constants existing (flagged explicitly in that service's code as `TODO[CRITICAL][REFUND]`).

## TODO: Verify (platform-level, not confirmed by this audit)

- Which service (if any) actually terminates Stripe's real HTTP webhook and republishes it onto `payment.webhook` for payments-ms to consume.
- Whether the root `.env` (present but gitignored) supplies all variables each service's Zod schema requires — the audits only checked `.env.example` files and source code, not the real `.env`.
- Production deployment process/CI — no CI config or production deployment scripts were found or reviewed as part of this audit.
