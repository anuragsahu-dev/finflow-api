# System Architecture

This document provides an overview of the FinFlow API's architecture, design decisions, and system components.

---

## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Technology Choices](#technology-choices)
3. [Application Structure](#application-structure)
4. [Module Architecture](#module-architecture)
5. [Authentication Flow](#authentication-flow)
6. [Payment System](#payment-system)
7. [Background Jobs](#background-jobs)
8. [CI/CD Pipeline](#cicd-pipeline)
9. [Security Considerations](#security-considerations)

---

## High-Level Architecture

### Production Infrastructure – AWS + Docker Swarm (3 Manager Nodes)

```
Client / Browser
       │
       ▼
   Internet (HTTPS)
       │
       ▼
┌──────────────────────── AWS Cloud (eu-north-1) ───────────────────────┐
│                                                                       │
│  ┌────────────── Docker Swarm Cluster (3 Manager Nodes) ────────────┐ │
│  │                                                                  │ │
│  │   Node Public Port :80 / :443                                    │ │
│  │            │                                                     │ │
│  │            ▼                                                     │ │
│  │      Swarm Routing Mesh                                          │ │
│  │   (any node can accept traffic)                                  │ │
│  │            │                                                     │ │
│  │            ▼                                                     │ │
│  │      Caddy Service (replicas:1)                                  │ │
│  │      Reverse Proxy + Auto TLS                                    │ │
│  │            │                                                     │ │
│  │            ▼                                                     │ │
│  │      Swarm Service VIP (app:3000)                                │ │
│  │            │                                                     │ │
│  │            ▼                                                     │ │
│  │   FinFlow App Service (replicas:N)                               │ │
│  │                                                                  │ │
│  │   Manager Node 1 (Leader) → Replica(s)                           │ │
│  │   Manager Node 2          → Replica(s)                           │ │
│  │   Manager Node 3          → Replica(s)                           │ │
│  │                                                                  │ │
│  │   Overlay Network: finflow_net                                   │ │
│  │   Docker Secrets (encrypted)                                     │ │
│  │                                                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┬──────────────┐
        ▼            ▼            ▼              ▼
  Neon PostgreSQL  Upstash Redis  AWS S3      Razorpay
   (Prisma ORM)     (BullMQ)     File Storage Payments
                      │
                      ▼
                  SMTP/Gmail
                   (Email)
```

> **How it works:** Caddy runs as a single replica. The app service can be scaled to N replicas
> (e.g. `docker service scale finflow-api_app=10`). Swarm distributes containers across the
> 3 manager nodes and Caddy's `reverse_proxy app:3000` automatically load balances via
> Swarm's **VIP (Virtual IP)**. The routing mesh ensures any node can accept incoming traffic
> on ports 80/443 even if Caddy isn't running on that specific node.

### Application Architecture – Internal Request Flow

```
  Client Request (HTTPS)
         │
         ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                   Caddy (Reverse Proxy)                         │
  │              TLS Termination (Let's Encrypt)                    │
  │              :443 → reverse_proxy app:3000                      │
  └──────────────────────────┬───────────────────────────────────────┘
                             │ HTTP :3000
                             ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                   Express 5 Application                         │
  │                                                                 │
  │  ┌────────────────────────────────────────────────────────────┐ │
  │  │                  Middleware Pipeline                        │ │
  │  │                                                            │ │
  │  │  Helmet → CORS → Cookie Parser → Rate Limiter (Redis)     │ │
  │  │       → JSON Parser → Request Logger                      │ │
  │  └────────────────────────┬───────────────────────────────────┘ │
  │                           │                                     │
  │  ┌────────────────────────▼───────────────────────────────────┐ │
  │  │                   API Router (/api/v1)                     │ │
  │  │                                                            │ │
  │  │  /auth    /users    /categories    /transactions           │ │
  │  │  /budgets /analytics /payments     /subscriptions          │ │
  │  │  /admin   /health                                          │ │
  │  └────────────────────────┬───────────────────────────────────┘ │
  │                           │                                     │
  │  ┌────────────────────────▼───────────────────────────────────┐ │
  │  │               Route-Level Middleware                       │ │
  │  │                                                            │ │
  │  │  Auth Middleware (JWT) → Zod Validation → Controller       │ │
  │  └────────────────────────┬───────────────────────────────────┘ │
  │                           │                                     │
  │  ┌────────────────────────▼───────────────────────────────────┐ │
  │  │              Controller → Service → Repository             │ │
  │  │                                                            │ │
  │  │  Controller: Extract req data, call service, send response │ │
  │  │  Service:    Business logic, orchestration, error handling │ │
  │  │  Repository: Prisma queries, DB error handling             │ │
  │  └────────────────────────┬───────────────────────────────────┘ │
  │                           │                                     │
  │  ┌────────────────────────▼───────────────────────────────────┐ │
  │  │                 Global Error Handler                       │ │
  │  │            ApiError → Standardized JSON Response           │ │
  │  └────────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────┘
         │                    │                     │
         ▼                    ▼                     ▼
  ┌─────────────┐    ┌──────────────┐      ┌──────────────┐
  │  PostgreSQL  │    │    Redis     │      │   BullMQ     │
  │  (Prisma)    │    │  (Rate Limit │      │  (Email,     │
  │  (Neon)      │    │   Sessions)  │      │  Scheduler)  │
  └─────────────┘    └──────────────┘      └──────────────┘
```

### CI/CD Pipeline – GitHub Actions to Production

```
  Developer Workstation
         │
         │  git push / PR
         ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                       GitHub Repository                         │
  │                                                                 │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  PR to main → ci.yaml                                    │   │
  │  │  ┌──────────┐ ┌────────────┐ ┌───────────────┐ ┌──────┐ │   │
  │  │  │   Lint   │ │ Unit Tests │ │ Integration   │ │Build │ │   │
  │  │  │ (3 Node  │ │ (Vitest)   │ │ Tests         │ │Check │ │   │
  │  │  │ versions)│ │            │ │ (PG + Redis)  │ │      │ │   │
  │  │  └──────────┘ └────────────┘ └───────────────┘ └──────┘ │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                           │ merge to main                       │
  │                           ▼                                     │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  Push to main → migrate.yaml                              │   │
  │  │  ┌────────────────────┐  ┌─────────────────────────────┐ │   │
  │  │  │ Migrate Production │  │ Sync Ephemeral DB (Neon #2) │ │   │
  │  │  │ DB (Neon #1)       │  │ (continue-on-error)         │ │   │
  │  │  └────────────────────┘  └─────────────────────────────┘ │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                           │ manual trigger                      │
  │                           ▼                                     │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  Manual → build.yaml                                      │   │
  │  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐ │   │
  │  │  │ Build Docker │  │ Tag image    │  │ Push to        │ │   │
  │  │  │ image (amd64)│  │ build-<SHA>  │  │ Docker Hub     │ │   │
  │  │  └──────────────┘  │ + latest     │  │                │ │   │
  │  │                    └──────────────┘  └────────────────┘ │   │
  │  │  Auto-updates compose.yaml with new image tag            │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                           │ manual trigger                      │
  │                           ▼                                     │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  Manual → deploy.yaml                                     │   │
  │  │  ┌─────────────┐  ┌───────────────┐  ┌────────────────┐ │   │
  │  │  │ SSH to EC2   │  │ docker stack  │  │ Health check   │ │   │
  │  │  │ Pull image   │  │ deploy        │  │ verification   │ │   │
  │  │  │              │  │ (rolling      │  │ (120s timeout) │ │   │
  │  │  │              │  │  update)      │  │                │ │   │
  │  │  └─────────────┘  └───────────────┘  └────────────────┘ │   │
  │  └──────────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │  Docker Swarm       │
                   │  (3 EC2 Managers)   │
                   │  Rolling Update     │
                   │  start-first        │
                   │  auto-rollback      │
                   └─────────────────────┘
```

### Production Stack

| Component         | Technology             | Purpose                               |
| ----------------- | ---------------------- | ------------------------------------- |
| **Reverse Proxy** | Caddy 2                | Automatic HTTPS, TLS certificates     |
| **Application**   | Node.js 22 + Express 5 | API server                            |
| **Orchestration** | Docker Swarm           | Container management, rolling updates |
| **Database**      | Neon (PostgreSQL)      | Serverless managed database           |
| **Cache/Queue**   | Upstash (Redis)        | Rate limiting, BullMQ job queue       |
| **Storage**       | AWS S3                 | Avatar uploads via pre-signed URLs    |
| **Payments**      | Razorpay               | Payment processing, webhooks          |

### Request Flow

1. Client sends HTTPS request to any Swarm node's public IP
2. Swarm routing mesh forwards traffic to the Caddy service
3. Caddy terminates TLS and proxies to the app via Swarm VIP (`app:3000`)
4. Swarm VIP load balances across available app replicas
5. Express handles routing, authentication, and validation
6. Service layer processes business logic
7. Repository layer interacts with PostgreSQL via Prisma
8. Response sent back through the same chain

---

## Technology Choices

### Core Stack

| Technology     | Purpose     | Reason                                            |
| -------------- | ----------- | ------------------------------------------------- |
| **Node.js 22** | Runtime     | Latest LTS, performance improvements              |
| **Express 5**  | Framework   | Mature, flexible, async/await support             |
| **TypeScript** | Language    | Type safety, better DX, fewer runtime errors      |
| **Prisma**     | ORM         | Type-safe queries, migrations, excellent DX       |
| **PostgreSQL** | Database    | ACID compliance, complex queries, reliability     |
| **Redis**      | Cache/Queue | Fast in-memory store for rate limiting and queues |

### Infrastructure

| Technology       | Purpose            | Reason                                          |
| ---------------- | ------------------ | ----------------------------------------------- |
| **Docker Swarm** | Orchestration      | Simple setup, built-in secrets, rolling updates |
| **Caddy**        | Reverse Proxy      | Automatic HTTPS with Let's Encrypt              |
| **Neon**         | Managed PostgreSQL | Serverless, auto-scaling, free tier             |
| **Upstash**      | Managed Redis      | Serverless, per-request pricing                 |

### Libraries

| Library                  | Purpose                                      |
| ------------------------ | -------------------------------------------- |
| **Zod**                  | Runtime validation with TypeScript inference |
| **BullMQ**               | Robust job queue with retries and scheduling |
| **Passport.js**          | Google OAuth authentication                  |
| **jsonwebtoken**         | JWT token generation and verification        |
| **Nodemailer + Mailgen** | Email sending with HTML templates            |
| **Razorpay SDK**         | Payment processing                           |
| **@aws-sdk/client-s3**   | AWS S3 operations                            |

---

## Application Structure

```
src/
├── config/                 # Configuration
│   ├── prisma.ts          # Database client singleton
│   ├── redis.ts           # Redis + BullMQ connections
│   ├── swagger.ts         # OpenAPI/Swagger config
│   ├── s3.ts              # AWS S3 client
│   └── env.ts             # Environment validation
│
├── middlewares/           # Express middlewares
│   ├── auth.middleware.ts # JWT verification
│   ├── validate.ts        # Zod request validation
│   ├── rateLimit.ts       # Redis-backed rate limiting
│   └── error.ts           # Global error handler
│
├── modules/               # Feature modules
│   ├── auth/              # OTP login, Google OAuth
│   ├── user/              # Profile, avatar upload
│   ├── category/          # System + custom categories
│   ├── transaction/       # Income/expense tracking
│   ├── budget/            # Monthly budgets + allocations
│   ├── analytics/         # Reports, trends, comparisons
│   ├── payment/           # Razorpay integration
│   ├── subscription/      # Plan management
│   ├── admin/             # Admin operations, pricing
│   ├── session/           # Multi-device sessions
│   ├── otp/               # OTP generation/verification
│   └── health/            # Liveness/readiness probes
│
├── infrastructure/        # External services
│   ├── payment/           # Razorpay provider
│   └── storage/           # S3 service
│
├── jobs/                  # Background processing
│   ├── queues/            # BullMQ queue definitions
│   └── workers/           # Email, scheduler workers
│
├── utils/                 # Utilities
│   ├── ApiError.ts        # Custom error class
│   ├── ApiResponse.ts     # Standardized responses
│   └── asyncHandler.ts    # Async error wrapper
│
├── types/                 # TypeScript definitions
├── app.ts                 # Express app setup
└── index.ts               # Server entry point
```

---

## Module Architecture

Each feature module follows a consistent layered pattern:

```
modules/<feature>/
├── <feature>.route.ts       # Route definitions + Swagger docs
├── <feature>.controller.ts  # HTTP handlers
├── <feature>.service.ts     # Business logic
├── <feature>.repository.ts  # Database operations
└── <feature>.validation.ts  # Zod schemas
```

### Layer Responsibilities

| Layer          | Responsibilities                                         |
| -------------- | -------------------------------------------------------- |
| **Route**      | Define endpoints, apply middlewares, Swagger docs        |
| **Controller** | Extract request data, call service, send response        |
| **Service**    | Business logic, orchestrate repositories, error handling |
| **Repository** | Database operations via Prisma, handle DB errors         |

### Key Principles

- Controllers have NO business logic
- Services don't know about HTTP (no req/res)
- Repositories don't know about business rules
- Each layer only talks to the layer below it

---

## Authentication Flow

### OTP-Based Passwordless Authentication

| Step | Action                                                    |
| ---- | --------------------------------------------------------- |
| 1    | User submits email to `POST /auth/start`                  |
| 2    | Server checks rate limit (Redis)                          |
| 3    | Generate 6-digit OTP, hash it, store in Redis (5 min TTL) |
| 4    | Queue email job via BullMQ                                |
| 5    | Email worker sends OTP to user                            |
| 6    | User submits OTP to `POST /auth/verify`                   |
| 7    | Server verifies OTP hash from Redis                       |
| 8    | Create session in database                                |
| 9    | Set HttpOnly cookies (accessToken, refreshToken)          |

### JWT Token Strategy

| Token             | Storage         | Expiry     | Purpose                 |
| ----------------- | --------------- | ---------- | ----------------------- |
| **Access Token**  | HttpOnly Cookie | 15 minutes | API authentication      |
| **Refresh Token** | HttpOnly Cookie | 7 days     | Obtain new access token |

### Google OAuth

Alternative login flow using Passport.js with Google OAuth 2.0 strategy.

---

## Payment System

See [PAYMENT_SYSTEM.md](PAYMENT_SYSTEM.md) for detailed documentation.

### Flow Summary

| Step | Endpoint                      | Action                                |
| ---- | ----------------------------- | ------------------------------------- |
| 1    | `GET /payments/plans`         | User views available plans            |
| 2    | `POST /payments/create-order` | Create Razorpay order                 |
| 3    | -                             | User pays via Razorpay checkout popup |
| 4    | `POST /payments/verify`       | Verify payment signature              |
| 5    | `POST /payments/webhook`      | Backup: Razorpay webhook              |
| 6    | -                             | Subscription activated                |

### Subscription Plans

| Plan            | Duration | Features                    |
| --------------- | -------- | --------------------------- |
| **Free**        | Forever  | Basic features              |
| **PRO Monthly** | 30 days  | All features                |
| **PRO Yearly**  | 365 days | All features (17% discount) |

---

## Background Jobs

### BullMQ Queues

| Queue       | Job Types             | Purpose                    |
| ----------- | --------------------- | -------------------------- |
| `email`     | `send-otp`, `welcome` | Email notifications        |
| `scheduler` | `check-expiry`        | Mark expired subscriptions |

### How It Works

1. Application adds job to queue (stored in Redis)
2. Worker picks up job from queue
3. Worker processes job (sends email, updates DB)
4. Failed jobs are retried automatically

---

## CI/CD Pipeline

### GitHub Actions Workflows

| Workflow       | Trigger      | Actions                                       |
| -------------- | ------------ | --------------------------------------------- |
| `ci.yaml`      | Pull Request | Lint, unit tests, integration tests, build    |
| `migrate.yaml` | Push to main | Run Prisma migrations on production DB        |
| `build.yaml`   | Manual       | Build Docker image, push to Docker Hub        |
| `deploy.yaml`  | Manual       | SSH to EC2, deploy Docker stack, health check |

### Deployment Flow

1. **PR created** → CI runs tests automatically
2. **PR merged** → Migrations run automatically
3. **Ready to deploy** → Manually trigger Build workflow
4. **Build complete** → Manually trigger Deploy workflow
5. **Deployed** → Health checks verify the application

### Docker Image

- Registry: Docker Hub
- Image: `as3305100/finflow-api`
- Tags: `build-<commit-sha>`, `latest`

---

## Security Considerations

### Implemented Security Measures

| Measure                     | Implementation                              |
| --------------------------- | ------------------------------------------- |
| **HTTPS**                   | Caddy automatic TLS with Let's Encrypt      |
| **CORS**                    | Restricted to specific origins              |
| **Helmet**                  | Security headers (XSS, clickjacking, etc.)  |
| **Rate Limiting**           | Redis-backed per-IP limits                  |
| **Input Validation**        | Zod schemas on all endpoints                |
| **SQL Injection**           | Prisma parameterized queries                |
| **CSRF Prevention**         | SameSite=Strict cookies                     |
| **OTP Hashing**             | Bcrypt hashed OTPs                          |
| **JWT in HttpOnly Cookies** | Not accessible via JavaScript               |
| **Docker Secrets**          | Sensitive data not in env vars (production) |

### Rate Limits

| Endpoint              | Limit        | Window     |
| --------------------- | ------------ | ---------- |
| `/auth/start`         | 5 requests   | 15 minutes |
| `/auth/verify`        | 10 requests  | 15 minutes |
| `/auth/refresh-token` | 30 requests  | 15 minutes |
| General API           | 100 requests | 15 minutes |

---

## Scalability Considerations

### Current Design

- Docker Swarm cluster with **3 Manager Nodes** on AWS EC2 (eu-north-1)
- Caddy runs as a single Docker Swarm service replica exposed through the Swarm routing mesh
- Docker Swarm schedules replicas across available nodes based on cluster state
- Overlay network (`finflow_net`) for inter-service communication
- Docker Secrets for production credentials
- Suitable for small to medium user base with high availability

### Future Scaling Path

| Component          | Scaling Strategy                                       |
| ------------------ | ------------------------------------------------------ |
| **Compute**        | Add more Docker Swarm nodes                            |
| **Database**       | Neon auto-scales, or migrate to RDS with read replicas |
| **Redis**          | Upstash auto-scales, or use Redis Cluster              |
| **Load Balancing** | Add AWS ALB in front of Swarm                          |
| **CDN**            | CloudFront for static assets                           |
