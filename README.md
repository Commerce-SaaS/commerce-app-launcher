# 🚀 Commerce App Launcher

A production-ready, microservices-based SaaS platform designed for e-commerce operations. This project orchestrates multiple independent services using Docker Compose, enabling scalable, modular commerce functionality with event-driven architecture.

**Status**: Enterprise-grade | **Stack**: NestJS | TypeORM | PostgreSQL | RabbitMQ | Redis

---

## 🎯 Project Overview

The Commerce App Launcher is the central orchestration hub for a distributed e-commerce platform. It coordinates multiple specialized microservices through an API Gateway, enabling seamless communication via REST APIs and asynchronous event-driven messaging.

**Key Capabilities:**
- 🔐 User authentication with JWT (access, refresh, reset tokens)
- 📦 Product catalog management
- 🏢 Multi-organization support
- 🛒 Order management & lifecycle
- 💳 Payment processing with Stripe integration
- 📧 Email notifications
- 📁 Media asset management with AWS S3
- ⚡ Real-time caching with Redis
- 🔄 Asynchronous task processing with RabbitMQ

This is **not** a standalone application—it's an orchestration platform where each service is independently deployable and scalable.

---

## 🏗️ System Architecture

The system follows a **microservices architecture** with event-driven patterns. Services communicate through:
- **Synchronous**: REST HTTP calls via API Gateway
- **Asynchronous**: RabbitMQ message queues for commands and events

```mermaid
graph TB
    Client["👤 Client Application<br/>http://localhost:3000"]
    
    subgraph Gateway["🌐 API Gateway Layer"]
        GW["Client Gateway<br/>:4000"]
    end
    
    subgraph CoreServices["🔧 Core Services"]
        AUTH["🔐 Auth MS<br/>:4002"]
        PRODUCTS["📦 Products MS<br/>:4001"]
        ORG["🏢 Organization MS<br/>:4003"]
        ORDERS["🛒 Orders MS<br/>:4007"]
    end
    
    subgraph ValueAddedServices["⭐ Value-Added Services"]
        PAYMENTS["💳 Payments MS<br/>:4006"]
        MEDIA["📁 Media MS<br/>:4004"]
        NOTIFICATIONS["📧 Notifications MS<br/>:4005"]
    end
    
    subgraph Infrastructure["📡 Infrastructure"]
        RMQ["🔄 RabbitMQ<br/>:5672 / 15672"]
        REDIS["⚡ Redis Cache<br/>:6379"]
    end
    
    subgraph Databases["🗄️ Data Layer"]
        AUTHDB["🔒 Auth DB<br/>:5433"]
        PRODUCTSDB["📦 Products DB<br/>:5432"]
        ORGDB["🏢 Organization DB<br/>:5434"]
        ORDERSDB["🛒 Orders DB<br/>:5436"]
        PAYMENTSDB["💳 Payments DB<br/>:5435"]
    end
    
    subgraph ExternalServices["🌍 External APIs"]
        STRIPE["💳 Stripe API"]
        AWS["☁️ AWS S3"]
        SMTP["📧 Email Provider"]
    end
    
    Client -->|REST API| GW
    
    GW -->|HTTP| AUTH
    GW -->|HTTP| PRODUCTS
    GW -->|HTTP| ORG
    GW -->|HTTP| ORDERS
    GW -->|HTTP| PAYMENTS
    
    AUTH --> AUTHDB
    PRODUCTS --> PRODUCTSDB
    ORG --> ORGDB
    ORDERS --> ORDERSDB
    PAYMENTS --> PAYMENTSDB
    
    AUTH -.->|Events| RMQ
    PRODUCTS -.->|Events| RMQ
    ORG -.->|Events| RMQ
    ORDERS -.->|Events| RMQ
    PAYMENTS -.->|Events| RMQ
    
    RMQ -.->|Consume| NOTIFICATIONS
    RMQ -.->|Consume| MEDIA
    
    NOTIFICATIONS -->|SMTP| SMTP
    MEDIA -->|SDK| AWS
    PAYMENTS -->|API| STRIPE
    
    GW -.->|Cache| REDIS
    AUTH -.->|Cache| REDIS
    PRODUCTS -.->|Cache| REDIS
    ORG -.->|Cache| REDIS
```

---

## 📦 Services Overview

### API Gateway
| Property | Value |
|----------|-------|
| **Service** | `client-gateway` |
| **Port** | `4000` |
| **Role** | Single entry point for all client requests |
| **Tech** | NestJS HTTP |
| **Database** | None (stateless) |
| **Dependencies** | Redis, RabbitMQ |

**Responsibilities:**
- Request routing to microservices
- Authentication middleware (JWT validation)
- Response aggregation
- Event publishing for client actions

---

### 🔐 Authentication microservice (`auth-ms`)
| Property | Value |
|----------|-------|
| **Port** | `4002` |
| **Database** | `authdb` (port 5433) |
| **Dependencies** | Redis, RabbitMQ |
| **External APIs** | Google OAuth |

**Responsibilities:**
- User registration & login
- JWT token generation (access, refresh, reset)
- Password reset flow
- Email verification
- Google OAuth integration
- Session management via Redis

**Key Features:**
- Refresh token rotation
- Email-based password recovery
- Google SSO support
- Templates for custom mailers

**Message Queues:**
- Publishes: `auth_queue` (commands), events to `events.notifications_queue`
- Consumes: User events for external integrations

---

### 📦 Products microservice (`products-ms`)
| Property | Value |
|----------|-------|
| **Port** | `4001` |
| **Database** | `productsdb` (port 5432) |
| **Dependencies** | Redis, RabbitMQ |
| **External APIs** | None |

**Responsibilities:**
- Product CRUD operations
- Inventory management
- Product categorization
- Stock level tracking
- Product pricing & variants

**Message Queues:**
- Publishes: `products_queue` (commands), events to `events.products_queue`
- Consumes: Order & payment events affecting stock

---

### 🏢 Organization microservice (`organization-ms`)
| Property | Value |
|----------|-------|
| **Port** | `4003` |
| **Database** | `organizationdb` (port 5434) |
| **Dependencies** | Redis, RabbitMQ |
| **External APIs** | None |

**Responsibilities:**
- Multi-tenancy support
- Organization creation & management
- Tenant configuration
- Team/workspace management
- Organization-level settings

**Message Queues:**
- Publishes: `organization_queue` (commands), events to `events.organization_queue` & `events.authz_queue`
- Consumes: Authorization and membership events

---

### 🛒 Orders microservice (`orders-ms`)
| Property | Value |
|----------|-------|
| **Port** | `4007` |
| **Database** | `ordersdb` (port 5436) |
| **Dependencies** | Redis, RabbitMQ |
| **External APIs** | None |

**Responsibilities:**
- Order creation & management
- Order status tracking
- Order history & fulfillment
- Inventory coordination
- Order analytics

**Message Queues:**
- Publishes: `orders_queue` (commands), events to `events.orders_queue`
- Consumes: Payment confirmations, inventory updates

**Typical Flow:**
1. Receives order creation request
2. Reserves inventory via Products MS
3. Publishes order.created event
4. Waits for payment confirmation
5. Updates order status

---

### 💳 Payments microservice (`payments-ms`)
| Property | Value |
|----------|-------|
| **Port** | `4006` |
| **Database** | `paymentsdb` (port 5435) |
| **Dependencies** | Redis, RabbitMQ |
| **External APIs** | Stripe API |

**Responsibilities:**
- Payment processing via Stripe
- Payment status tracking
- Invoice generation
- Refund handling
- Webhook validation (Stripe webhooks)
- Connect account management

**Message Queues:**
- Publishes: `payments_queue` (commands), events to `events.payments_queue`
- Consumes: Order events, payment intents

**Stripe Integration:**
- Processes payment intents
- Handles webhooks for payment completion
- Supports Stripe Connect for multi-vendor scenarios

---

### 📁 Media microservice (`media-ms`)
| Property | Value |
|----------|-------|
| **Port** | `4004` |
| **Database** | None (AWS S3 based) |
| **Dependencies** | RabbitMQ |
| **External APIs** | AWS S3 |

**Responsibilities:**
- Media file uploads & storage
- AWS S3 integration
- Image resizing & optimization
- File metadata tracking
- CDN integration support

**Message Queues:**
- Publishes: `media_queue` (commands)
- Consumes: Product/avatar upload events

**Features:**
- Direct S3 integration via SDK
- Temporary signed URLs for downloads
- Support for product images, avatars, documents

---

### 📧 Notifications microservice (`notifications-ms`)
| Property | Value |
|----------|-------|
| **Port** | `4005` |
| **Database** | None (stateless, read-only logs) |
| **Dependencies** | RabbitMQ |
| **External APIs** | SMTP (Email provider) |

**Responsibilities:**
- Email sending
- Notification templating
- Email event logging
- Batch notification processing

**Message Queues:**
- Subscribes to: `events.notifications_queue` (all notification events)

**Triggered By:**
- User registration (welcome email)
- Password reset (reset link)
- Order created (order confirmation)
- Payment received (receipt)
- Organization invites
- Custom notifications

**Email Provider:**
- SMTP configuration (Gmail, SendGrid, etc.)
- Handlebars templating support

---

## 🔄 Service Communication

### Synchronous (Request-Response)
All client requests flow through the **Client Gateway** which internally calls microservices via HTTP.

**Example Flows:**

```
Client Request:          POST /api/products
                         ↓
                    Client Gateway
                         ↓
                    Products MS
                         ↓
                    Response → Client
```

### Asynchronous (Event-Driven via RabbitMQ)

Services emit events to RabbitMQ for inter-service communication. This enables loose coupling and handles long-running processes.

#### Command Queues (1-to-1)
- `auth_queue` → Auth MS commands
- `products_queue` → Products MS commands
- `organization_queue` → Organization MS commands
- `orders_queue` → Orders MS commands
- `payments_queue` → Payments MS commands
- `media_queue` → Media MS commands
- `client_gateway_queue` → Gateway commands

#### Event Queues (1-to-many)
- `events.notifications_queue` ← Auth, Orders, Payments (fan-out to Notifications MS)
- `events.organization_queue` ← Organization MS → Authz systems
- `events.orders_queue` ← Orders MS → Analytics/Reporting
- `events.payments_queue` ← Payments MS → Orders MS (confirmation)
- `events.products_queue` ← Products MS → Inventory tracking
- `events.authz_queue` ← Auth/Organization MS → Authorization systems

#### Example Event Flow:

```
1. User places order
   Order MS → RabbitMQ: { type: 'order.created', orderId: '123' }
   
2. Multiple subscribers receive event:
   a) Payments MS: Prepares for payment
   b) Notifications MS: Sends order confirmation email
   c) Products MS: Reserves inventory
   
3. Payment confirmed
   Payments MS → RabbitMQ: { type: 'payment.confirmed', orderId: '123' }
   
4. Orders MS receives and updates status
   Orders MS: status = 'confirmed'
```

### Caching Layer (Redis)
- **Session Storage**: Auth tokens & user sessions
- **Data Caching**: Frequently accessed products, organizations
- **Rate Limiting**: API rate limiting per user/IP
- **Cache Key Patterns**: `{service}:{entity}:{id}` (e.g., `products:product:5`)

---

## 🔐 Authentication Flow

### 1. User Registration/Login

```
Client
  ↓ (email + password)
Auth MS
  ├─ Validate credentials
  ├─ Generate JWT tokens:
  │  ├─ access_token (short-lived: 15min)
  │  ├─ refresh_token (long-lived: 7days)
  │  └─ reset_token (for password reset)
  ├─ Store refresh_token in Redis
  └─ Emit: user.created → notifications_queue
       → Sends welcome email
```

### 2. Request Authentication
```
Client Request + access_token
  ↓
Client Gateway (middleware)
  ├─ Validate JWT signature
  ├─ Check token expiration
  ├─ Extract user claims
  └─ Forward to microservice
     OR
     (if expired) → Use refresh_token to get new access_token

Microservice
  ├─ Verify user has required permissions
  ├─ Process request
  └─ Return response
```

### 3. Token Refresh Flow
```
Client (access_token expired)
  ↓
Auth MS: POST /auth/refresh
  ├─ Validate refresh_token in Redis
  ├─ Check for token rotation rules
  ├─ Issue new access_token
  └─ Return new token pair
```

### 4. Password Reset Flow
```
Client: POST /auth/forgot-password
  ↓
Auth MS
  ├─ Generate reset_token (short-lived)
  ├─ Store in Redis with TTL (15min)
  ├─ Send reset email via notifications_queue
  │   Email includes: /reset-password?token=xyz
  │
Client: POST /auth/reset-password (with reset_token)
  ├─ Validate reset_token
  ├─ Hash new password
  ├─ Update in database
  └─ Invalidate all sessions in Redis
```

### 5. Google OAuth
```
Client → Google OAuth Consent
  ↓
Client redirects to Auth MS: /auth/google/callback
  ├─ Verify Google token
  ├─ Find or create user
  ├─ Issue JWT tokens
  └─ Redirect to frontend with tokens
```

**Security Features:**
- ✅ Refresh token rotation (invalidates old token)
- ✅ Password hashing with bcrypt
- ✅ Token blacklisting on logout
- ✅ CORS validation
- ✅ Rate limiting on auth endpoints
- ✅ Email verification before login
- ✅ Session invalidation on password change

---

## ⚙️ Environment Configuration

### Global Environment Variables (`.env`)

```bash
# Node Environment
NODE_ENV=development|production|staging

# RABBITMQ
RABBITMQ_USER=user
RABBITMQ_PASS=password
RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_URL=amqp://user:password@rabbitmq:5672

# REDIS
REDIS_HOST=redis
REDIS_PORT=6379

# CLIENT URLS
FRONT_URL=http://localhost:3000        # Frontend application URL
CLIENT_URL=http://localhost:3000       # Client for payments/webhooks

# JWT SECRETS (generate strong values!)
JWT_SECRET_ACCESS=your_access_secret   # Access token (15 min)
JWT_SECRET_REFRESH=your_refresh_secret # Refresh token (7 days)
JWT_SECRET_RESET=your_reset_secret     # Password reset (15 min)

# STRIPE
STRIPE_SECRET=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_CONNECT_WEBHOOK_SECRET=whsec_...

# GOOGLE OAUTH
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com

# SMTP (Email)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_USER=your_email@gmail.com
SMTP_PASS=app_password
SMTP_FROM="Your App <noreply@example.com>"

# AWS S3
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=us-east-1
AWS_BUCKET=your-bucket-name

# DATABASE
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_secure_password
DB_PORT=5432

PRODUCTS_DB=productsdb
AUTH_DB=authdb
ORGANIZATION_DB=organizationdb
ORDERS_DB=ordersdb
PAYMENTS_DB=paymentsdb

# MESSAGE QUEUES (command/topic)
RABBITMQ_QUEUE_CLIENT_GATEWAY=client_gateway_queue
RABBITMQ_QUEUE_AUTH=auth_queue
RABBITMQ_QUEUE_ORGANIZATION=organization_queue
RABBITMQ_QUEUE_ORDERS=orders_queue
RABBITMQ_QUEUE_PAYMENTS=payments_queue
RABBITMQ_QUEUE_PRODUCTS=products_queue
RABBITMQ_QUEUE_MEDIA=media_queue

# MESSAGE QUEUES (events)
RMQ_EVENTS_QUEUE_NOTIFICATIONS=events.notifications_queue
RMQ_EVENTS_QUEUE_ORGANIZATION=events.organization_queue
RMQ_EVENTS_QUEUE_ORDERS=events.orders_queue
RMQ_EVENTS_QUEUE_PAYMENTS=events.payments_queue
RMQ_EVENTS_QUEUE_PRODUCTS=events.products_queue
RMQ_EVENTS_QUEUE_AUTHZ=events.authz_queue
```

**Important Security Notes:**
- 🔒 Never commit `.env` with real secrets
- 🔒 Use strong, random values for JWT secrets (min 32 chars)
- 🔒 Rotate secrets periodically
- 🔒 Use environment-specific configurations
- 🔒 Store secrets in secure vault (HashiCorp Vault, AWS Secrets Manager)

---

## 🐳 Docker & Orchestration

### Network Architecture

```
┌─────────────────────────────────────────────┐
│         Docker Network: app-network         │
├─────────────────────────────────────────────┤
│ All services communicate through this network│
│ Services are discoverable by hostname       │
│ (e.g., rabbitmq:5672, redis:6379)          │
└─────────────────────────────────────────────┘
```

### Service Dependencies

**Service startup order (automatic with `depends_on`):**

1. **Infrastructure First**
   - Redis (no dependencies)
   - RabbitMQ with healthcheck (30s startup)

2. **Databases** (parallel)
   - auth-db
   - products-db
   - organization-db
   - orders-db
   - payments-db

3. **Gateway & Microservices** (parallel, wait for dependencies)
   - client-gateway (waits: RabbitMQ healthy + Redis started)
   - auth-ms (waits: auth-db + RabbitMQ + Redis)
   - products-ms (waits: products-db + RabbitMQ + Redis)
   - organization-ms (waits: organization-db + RabbitMQ + Redis)
   - orders-ms (waits: orders-db + RabbitMQ + Redis)
   - payments-ms (waits: payments-db + RabbitMQ + Redis)

4. **Utility Services** (parallel, wait for RabbitMQ)
   - media-ms (waits: RabbitMQ)
   - notifications-ms (waits: RabbitMQ)

### Container Health Monitoring

**RabbitMQ Healthcheck:**
```yaml
healthcheck:
  test: ["CMD", "rabbitmq-diagnostics", "check_running"]
  interval: 10s
  timeout: 5s
  retries: 10
  start_period: 10s
```

**Service Volumes:**
- Each microservice: `./service-name:/usr/src/app` (live reload)
- Each database: `./service-name/postgres:/var/lib/postgresql/data` (data persistence)
- Redis: `redis-data:/data` (named volume)

### Port Mappings

| Service | Container Port | Host Port |
|---------|----------------|-----------|
| Client Gateway | 4000 | 4000 |
| Products MS | 4001 | 4001 |
| Auth MS | 4002 | 4002 |
| Organization MS | 4003 | 4003 |
| Media MS | 4004 | 4004 |
| Notifications MS | 4005 | 4005 |
| Payments MS | 4006 | 4006 |
| Orders MS | 4007 | 4007 |
| RabbitMQ AMQP | 5672 | 5672 |
| RabbitMQ Dashboard | 15672 | 15672 |
| Auth DB | 5432 | 5433 |
| Products DB | 5432 | 5432 |
| Organization DB | 5432 | 5434 |
| Payments DB | 5432 | 5435 |
| Orders DB | 5432 | 5436 |
| Redis | 6379 | 6379 |

---

## 🚀 Getting Started

### Prerequisites

- **Docker** >= 20.10
- **Docker Compose** >= 2.0
- **Node.js** >= 18 (for local development without Docker)
- **Git**

### 1. Clone & Setup

```bash
# Clone repository
git clone <repository-url>
cd commerce-app-launcher

# Copy environment template
cp .env.example .env

# Edit .env with your secrets
#⚠️ Important: Set strong JWT secrets!
# ⚠️ Configure Stripe, Google OAuth, SMTP, AWS credentials
```

### 2. Start All Services

```bash
# Build and start everything
docker compose up --build

# Output should show all services starting:
# ✓ redis
# ✓ rabbitmq
# ✓ auth-db (+ other DBs)
# ✓ client-gateway
# ✓ auth-ms
# (all other services...)
```

### 3. Verify Services

```bash
# Check all containers are running
docker compose ps

# Verify API Gateway is responding
curl http://localhost:4000/health

# Check RabbitMQ Dashboard
# Visit: http://localhost:15672 (user: guest, pass: guest)

# Test a service directly
curl http://localhost:4002/health  # Auth MS health
```

### 4. Database Initialization

```bash
# Databases are automatically initialized on first run
# TypeORM migrations run automatically:
# - Each service connects to its database
# - Synchronizes schema based on entities
# - Creates tables/indexes

# Check database connections
docker logs auth-ms | grep "connected"
docker logs products-ms | grep "connected"
```

### 5. Access Services

| Service | URL | Port |
|---------|-----|------|
| Client Gateway | `http://localhost:4000` | 4000 |
| API Gateway Docs | `http://localhost:4000/api` | - |
| RabbitMQ Dashboard | `http://localhost:15672` | 15672 |
| Auth MS | `http://localhost:4002` | 4002 |
| Products MS | `http://localhost:4001` | 4001 |
| (Other Services) | See port mappings | - |

---

## 🧪 Development Workflow

### Local Development vs Docker

#### Option A: Docker (Recommended for Full Stack)
```bash
# All services in containers with live reload
docker compose up

# Monitor logs
docker compose logs -f

# Restart a specific service
docker compose restart auth-ms

# Stop everything
docker compose down
```

#### Option B: Hybrid (Service in IDE + Docker Infrastructure)
```bash
# Start infrastructure only (no Node services)
docker compose up redis rabbitmq auth-db products-db organization-db orders-db payments-db media-ms notifications-ms

# In VS Code, run specific service:
cd auth-ms
npm install
npm run start:dev

# The service connects to Docker infrastructure
```

### Viewing Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f auth-ms

# Last 100 lines
docker compose logs -f --tail=100 products-ms

# Real-time with timestamps
docker compose logs -f --timestamps
```

### Debugging

#### VSCode Debugging
```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Auth MS Debug",
      "program": "${workspaceFolder}/auth-ms/node_modules/.bin/nest",
      "args": ["start", "--debug", "--watch"],
      "console": "integratedTerminal",
      "internalConsoleOptions": "openOnSessionStart"
    }
  ]
}
```

#### Access Pod Logs
```bash
# SSH into container
docker compose exec auth-ms bash

# Run commands
npm run test
npm run lint

# Check environment
env | grep RABBITMQ
```

### Running Tests

```bash
# Run tests in specific service
docker compose exec products-ms npm run test

# Watch mode
docker compose exec auth-ms npm run test:watch

# Coverage
docker compose exec orders-ms npm run test:cov

# E2E tests
docker compose exec payments-ms npm run test:e2e
```

### Database Management

```bash
# Access database directly
docker compose exec auth-db psql -U postgres -d authdb

# Common SQL commands
# \dt → list tables
# \d users → describe table
# SELECT * FROM users;

# Backup database
docker compose exec auth-db pg_dump -U postgres authdb > backup.sql

# Restore database
docker compose exec -T auth-db psql -U postgres authdb < backup.sql
```

### Cleaning Up

```bash
# Stop all services
docker compose down

# Remove containers + volumes (WARNING: deletes data!)
docker compose down -v

# Remove only named volumes
docker volume rm commerce-app-launcher_redis-data

# Prune unused Docker resources
docker system prune -f
```

---

## 🔁 Real-World Flow Examples

### Example 1: User Registration & Welcome Email

```
1. Client → POST /api/auth/register
   { email: "user@example.com", password: "..." }
                    ↓
2. Client Gateway routes to Auth MS
                    ↓
3. Auth MS
   ├─ Validate input (zod/class-validator)
   ├─ Hash password (bcrypt)
   ├─ Store in authdb
   ├─ Generate JWT tokens
   ├─ Store refresh_token in Redis
   ├─ Emit: user.created event
   │   { userId: "123", email: "user@example.com" }
   │   → Published to events.notifications_queue
   └─ Return access_token to client
                    ↓
4. Notifications MS receives event
   ├─ Render email template (Handlebars)
   ├─ Send via SMTP
   └─ Log delivery status
                    ↓
5. User receives: "Welcome to Commerce App!"
```

### Example 2: Product Order Lifecycle

```
Phase 1: Create Order
─────────────────────
Client → POST /api/orders
{ productIds: ["1", "2"], quantity: [1, 2] }
        ↓
Client Gateway → Orders MS
        ↓
Orders MS
├─ Validate inventory with Products MS (HTTP call)
├─ Reserve inventory
├─ Create order record (ordersdb)
├─ Set status = pending_payment
├─ Emit: order.created
│   → events.orders_queue (analytics)
│   → events.notifications_queue (send email)
└─ Return orderId, sessionId to client


Phase 2: Process Payment
────────────────────────
Client → POST /api/payments/create-session
{ orderId: "order-123" }
        ↓
Client Gateway → Payments MS
        ↓
Payments MS
├─ Fetch order from Orders MS (HTTP)
├─ Create Stripe Session
├─ Emit: payment.session.created
└─ Return sessionId to client
        ↓
Client redirects to Stripe Checkout


Phase 3: Payment Confirmation (Webhook)
────────────────────────────────────────
Stripe → POST /api/payments/stripe-webhook
{ type: "checkout.session.completed", ... }
        ↓
Payments MS
├─ Verify webhook signature
├─ Validate Stripe session completed
├─ Update payment record (paymentsdb)
├─ Emit: payment.confirmed { orderId: "order-123" }
│   → events.payments_queue → Orders MS
│   → events.notifications_queue → Send receipt
│   → events.products_queue → Deduct inventory
└─ Return 200 OK to Stripe


Phase 4: Order Fulfillment
───────────────────────────
Orders MS receives: payment.confirmed event
├─ Update order status = confirmed
├─ Emit: order.confirmed
│   → events.notifications_queue (send confirmation email)
└─ Database updated (ordersdb)

Notifications MS receives: order.confirmed
├─ Render: "Order Confirmed" email
├─ Include: Order number, tracking link, items
├─ Send email
└─ Log delivery


Phase 5: Customer Receives Email
─────────────────────────────────
✅ Welcome email + payment receipt + order confirmation
```

### Example 3: Organization Management (Multi-Tenancy)

```
Create Organization
───────────────────
Client → POST /api/organizations
{ name: "Acme Corp", adminId: "user-123" }
        ↓
Client Gateway → Organization MS
        ↓
Organization MS
├─ Validate owner permissions
├─ Create organization (organizationdb)
├─ Set owner as admin
├─ Emit: organization.created
│   → events.organization_queue
│   → events.authz_queue (authorization service)
└─ Return organizationId
        ↓
Authorization service updates permissions:
├─ Grant admin role to owner
├─ Create default roles (admin, member, viewer)
└─ Sync permissions cache (Redis)


Add Member to Organization
──────────────────────────
Client → POST /api/organizations/org-123/members
{ email: "john@example.com", role: "member" }
        ↓
Organization MS
├─ Verify requester is admin
├─ Create membership record
├─ Emit: member.invited
│   → events.notifications_queue
└─ Return membership details
        ↓
Notifications MS
├─ Send: "John, you've been invited to Acme Corp"
├─ Include: Acceptance link with token
└─ John clicks link → Accepts membership → Added to org
```

---

## 📌 Important Notes for Developers

### 1. **Service Independence**
- ✅ Each service has its **own database** (no shared database)
- ✅ Services are **independently deployable**
- ❌ Do NOT query another service's database directly
- ✅ Use HTTP APIs or RabbitMQ for inter-service communication

### 2. **Message Queue Patterns**

**Command Queues** (service-specific):
- Slower, ordered processing
- Example: `auth_queue` → only Auth MS consumes
- Use for: Specific tasks for a service

**Event Queues** (broadcast):
- Multiple subscribers allowed
- Example: `events.notifications_queue` ← many services publish
- Use for: Notifications, analytics, triggering workflows

### 3. **Database Migrations**
- TypeORM synchronizes schema automatically in dev
- For production, use migrations:
  ```bash
  npm run migration:generate src/migrations/CreateUsersTable
  npm run migration:run
  npm run migration:revert
  ```

### 4. **Error Handling**

```typescript
// Services should emit error events for failed operations
// Example: Payment failed
try {
  await stripe.createPaymentIntent(...);
} catch (error) {
  this.rabbitMQ.publish('events.payments_queue', {
    type: 'payment.failed',
    orderId,
    reason: error.message
  });
  throw error; // Let gateway handle 500 response
}
```

### 5. **Caching Strategy**

```typescript
// Products MS caching
const cachedProduct = await redis.get(`products:product:${id}`);
if (cachedProduct) return JSON.parse(cachedProduct);

const product = await this.db.findProduct(id);
await redis.setex(`products:product:${id}`, 3600, JSON.stringify(product)); // 1 hour TTL
return product;

// Invalidate on update
await redis.del(`products:product:${id}`);
```

### 6. **Rate Limiting**
- Configured per service
- Uses Redis-based tracking
- Protects auth endpoints especially

### 7. **Logging & Monitoring**
- Use NestJS Logger in each service
- Centralized logs via Docker logging driver
- Recommended: ELK Stack (Elasticsearch, Logstash, Kibana) in production

### 8. **Environment-Specific Configs**

```bash
# Development
NODE_ENV=development → auto reload, detailed logs, weak secrets OK

# Staging
NODE_ENV=staging → real credentials, medium logging

# Production
NODE_ENV=production → optimized, security strict, minimal logging
```

### 9. **Adding New Microservice**

1. Create new folder: `service-name-ms/`
2. Copy structure from existing service (e.g., copy `products-ms/`)
3. Update database name in `.env`
4. Add to `docker-compose.yml`:
   ```yaml
   new-service-ms:
     build: ./new-service-ms
     ports:
       - "4008:4008"  # Use next available port
     environment:
       PORT: 4008
       DB_HOST: new-service-db
       # ... other vars
     depends_on:
       rabbitmq:
         condition: service_healthy
   ```
5. Run: `docker compose up --build`

### 10. **Performance Considerations**
- ✅ Use Redis for session caching (avoid DB hits per request)
- ✅ Batch RabbitMQ messages when processing events
- ✅ Implement database indexes on frequently queried fields
- ✅ Use connection pooling (TypeORM default: 10 connections)
- ✅ Monitor: RabbitMQ queue depths, Redis memory usage, DB connections

---

## 📚 Additional Resources

### Documentation
- [NestJS Official](https://docs.nestjs.com/)
- [TypeORM Docs](https://typeorm.io/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)
- [Docker Compose Spec](https://docs.docker.com/compose/compose-file/)

### Production Checklist
- [ ] All environment variables in secure vault
- [ ] HTTPS enabled for all services
- [ ] Database backups configured
- [ ] Monitoring & alerting setup (Prometheus, Grafana)
- [ ] Centralized logging (ELK Stack)
- [ ] API rate limiting on all endpoints
- [ ] CORS properly configured
- [ ] Input validation on all endpoints
- [ ] Database connection pooling optimized
- [ ] RabbitMQ persistence enabled
- [ ] Redis replication setup

---

## 🤝 Contributing

When modifying this orchestration project:

1. Update docker-compose.yml for infrastructure changes
2. Document new environment variables in `.env.example`
3. Update this README for architecture changes
4. Test full stack: `docker compose down -v && docker compose up --build`
5. Verify all services are healthy before committing

---

## 📝 License

Proprietary - All rights reserved.

---

**Last Updated**: April 2026 | **Maintainers**: Fernando Ituarte