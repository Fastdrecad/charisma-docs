**Welcome to Charisma AI Backend!** 🚀

This comprehensive guide will help you get up to speed with the backend codebase, architecture, and development workflows.

---

## Next Steps

- **Deployment:** See [`DEPLOYMENT.md`](./DEPLOYMENT.md) for how to ship changes safely.
- **Backend Tech Stack:** See [`BACKEND_TECH_STACK_COMPLETE.md`](./BACKEND_TECH_STACK_COMPLETE.md) for a full overview of technologies and versions.
- **Controller Pattern Standard:** See [`CONTROLLER_PATTERN_STANDARD.md`](./CONTROLLER_PATTERN_STANDARD.md) for the standardized controller architecture.
- **Magic Opener feature guide:** See [`MAGIC_OPENERS_BACKEND_GUIDE.md`](./magic/MAGIC_OPENERS_BACKEND_GUIDE.md) to deep dive into the Magic Opener module.
- **Sessions onboarding:** See [`SESSIONS_BACKEND_ONBOARDING.md`](./sessions/SESSIONS_BACKEND_ONBOARDING.md) for a guided tour of the Sessions module.

---

## 📋 Table of Contents

1. [Quick Start](#quick-start)
2. [Project Overview](#project-overview)
3. [Architecture & Structure](#architecture--structure)
4. [Core Modules Deep Dive](#core-modules-deep-dive)
5. [Feature Modules](#feature-modules)
6. [Development Workflow](#development-workflow)
7. [Testing](#testing)
8. [Database & Migrations](#database--migrations)
9. [API Documentation](#api-documentation)
10. [Deployment](#deployment)
11. [Troubleshooting](#troubleshooting)
12. [Next Steps](#next-steps)

---

## 🚀 Quick Start

### Prerequisites

- **Node.js** 20+ (check: `node --version`)
- **npm** or **yarn**
- **Docker** & **Docker Compose** (for local services)
- **Git**
- **Doppler CLI** (optional, for secrets management)

### 5-Minute Setup

```bash
# 1. Clone repository
git clone <repository-url>
cd charisma/backend

# 2. Install dependencies
npm install

# 3. Setup environment (choose one method)

# Option A: Local development with Neon (RECOMMENDED)
cp .env.local.example .env.local
# Edit .env.local with your Neon development branch connection string

# Option B: Docker development
cp .env.docker.example .env.docker
# Edit .env.docker with your Clerk credentials

# 4. Start development server
npm run dev:local    # Option A (Neon)
# OR
npm run dev          # Option B (Docker)

# 5. Verify it's working
curl http://localhost:8080/health
```

**Expected Output:**

```json
{
  "success": true,
  "data": {
    "status": "ok",
    "timestamp": "2025-01-27T10:00:00.000Z",
    "environment": "development"
  }
}
```

✅ **You're ready to code!**

---

## 📖 Project Overview

### What is Charisma AI?

Charisma AI is a **dating coach mobile application** that helps users improve their dating conversations through AI-powered suggestions and analysis.

### Tech Stack

| Category           | Technology                       | Version |
| ------------------ | -------------------------------- | ------- |
| **Runtime**        | Node.js                          | 20+     |
| **Language**       | TypeScript                       | 5.9+    |
| **Framework**      | Express.js                       | 5.1+    |
| **Database**       | PostgreSQL (Neon)                | 16      |
| **ORM**            | Prisma                           | 6+      |
| **Cache/Queue**    | Redis (Upstash)                  | 7       |
| **Job Queue**      | BullMQ                           | Latest  |
| **Authentication** | Clerk                            | Latest  |
| **Storage**        | Cloudflare R2                    | Latest  |
| **AI Providers**   | OpenAI, Anthropic, Google Gemini | Latest  |
| **Testing**        | Jest + Supertest                 | Latest  |
| **Logging**        | Pino                             | Latest  |

### Key Features

1. **Magic Openers** - AI-generated conversation starters
2. **Sessions** - Screenshot analysis with OCR and AI suggestions
3. **RAG System** - Vector similarity search for contextual advice
4. **Phase Detection** - Conversation phase analysis (1-4)
5. **Real-time Updates** - WebSocket streaming for AI generation
6. **User Management** - Profile, onboarding, preferences

---

## 🏗️ Architecture & Structure

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Client (React Native)                │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTP/WebSocket
┌──────────────────────▼──────────────────────────────────┐
│              Backend API (Express.js)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Routes     │  │ Controllers  │  │  Services    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                  │           │
│  ┌──────▼─────────────────▼──────────────────▼───────┐  │
│  │         Core Infrastructure                       │  │
│  │  • Logger (Pino)  • Error Handling  • Auth     │  │
│  │  • WebSocket      • Queue (BullMQ)  • Database │  │
│  └──────────────────────────────────────────────────┘  │
└──────┬──────────────┬──────────────┬──────────────┬─────┘
       │              │              │              │
┌──────▼──────┐ ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐
│ PostgreSQL  │ │   Redis   │ │ Cloudflare│ │   AI APIs │
│   (Neon)    │ │ (Upstash) │ │     R2    │ │ (OpenAI,  │
│             │ │           │ │           │ │ Gemini)   │
└─────────────┘ └───────────┘ └───────────┘ └───────────┘
```

### Project Structure

```
backend/
├── src/
│   ├── server.ts              # Application entry point
│   ├── app.ts                 # Express app factory
│   │
│   ├── core/                  # Core infrastructure
│   │   ├── logger/            # Pino logger configuration
│   │   ├── env/               # Environment validation (Zod)
│   │   ├── error/             # Error classes & catalog
│   │   ├── middleware/        # Express middleware
│   │   ├── database/          # Prisma client
│   │   ├── queue/             # Redis/BullMQ connection
│   │   └── websocket/         # WebSocket server
│   │
│   ├── modules/               # Business domain modules
│   │   ├── user/              # User management
│   │   ├── webhook/           # Webhook handling (Clerk)
│   │   ├── magic-opener/      # Magic Openers feature
│   │   ├── sessions/          # Sessions feature (OCR, RAG, AI)
│   │   ├── ai/                # AI client & strategy
│   │   ├── analytics/         # Cost tracking
│   │   └── notification/      # Push notifications
│   │
│   ├── jobs/                  # Scheduled jobs
│   │   └── scheduler.ts      # Job scheduler
│   │
│   └── routes/                # Global routes
│       └── test.routes.ts     # Dev/test routes
│
├── prisma/
│   ├── schema.prisma          # Database schema
│   └── migrations/            # Migration files
│
├── docs/                      # Documentation
├── .env.local.example          # Environment template
└── package.json
```

### Design Patterns

1. **Dependency Injection** - Services receive dependencies via factory functions
2. **Factory Pattern** - All services use `create*Service()` pattern
3. **Repository Pattern** - Prisma acts as data access layer
4. **Strategy Pattern** - AI provider selection with fallbacks
5. **Service Layer** - Business logic separated from controllers

---

## 🔧 Core Modules Deep Dive

### 1. Environment Configuration (`src/core/env/`)

**Purpose:** Type-safe environment variable validation using Zod.

**Key File:** `src/core/env/index.ts`

```typescript
// All environment variables are validated at startup
export const env = envSchema.parse(process.env);

// Usage in code
import { env } from "./core/env/index.js";
const dbUrl = env.DATABASE_URL; // Type-safe!
```

**Required Variables:**

- `DATABASE_URL` - PostgreSQL connection string
- `REDIS_URL` - Redis connection string
- `CLERK_SECRET_KEY` - Clerk authentication secret
- `CLERK_WEBHOOK_SECRET` - Clerk webhook verification

**Optional Variables:**

- `GEMINI_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` - AI providers
- `R2_*` - Cloudflare R2 storage credentials
- `AI_PROVIDER_STRATEGY_PRIMARY` - Primary AI provider (default: `gpt-4o`)

### 2. Logger (`src/core/logger/`)

**Purpose:** Structured logging with Pino.

**Usage:**

```typescript
import logger from "./core/logger/index.js";

logger.info({ userId, action: "login" }, "User logged in");
logger.error({ error }, "Failed to process request");
logger.debug({ sessionId }, "Processing session");
```

**Log Levels:** `fatal`, `error`, `warn`, `info`, `debug`, `trace`

### 3. Error Handling (`src/core/error/`)

**Purpose:** Centralized error handling with custom error catalog.

**Key Files:**

- `AppError.ts` - Base error class
- `error.catalog.ts` - Error code definitions
- `errorHandler.ts` - Express error middleware

**Usage:**

```typescript
import { AppError } from "./core/error/AppError.js";
import { ErrorCodes } from "./core/error/error.catalog.js";

throw new AppError(
  "User not found",
  404,
  ErrorCodes.USER_NOT_FOUND,
  true // isOperational
);
```

### 4. Database (`src/core/database/`)

**Purpose:** Prisma client singleton with connection pooling.

**Key File:** `src/core/database/client.ts`

```typescript
import { prisma } from "./core/database/client.js";

// Use Prisma client
const user = await prisma.user.findUnique({ where: { id } });
```

**Schema Location:** `prisma/schema.prisma`

### 5. WebSocket (`src/core/websocket/`)

**Purpose:** Real-time communication for AI streaming and status updates.

**Key Features:**

- AI generation streaming (Magic Openers)
- Session status updates (OCR progress)
- Rate limiting per connection

**Usage:**

```typescript
import { publishStatusUpdate } from "./core/websocket/websocket-server.js";

await publishStatusUpdate(sessionId, "analysis_in_progress");
```

### 6. Queue (`src/core/queue/`)

**Purpose:** BullMQ connection management for background jobs.

**Used For:**

- OCR processing (async)
- Long-running AI operations
- Scheduled tasks

---

## 🎯 Feature Modules

### Module Structure Pattern

Every feature module follows this structure:

```
modules/[feature]/
├── [feature].service.ts      # Business logic
├── [feature].controller.ts   # HTTP handlers
├── [feature].routes.ts       # Express routes
├── [feature].validation.ts   # Request validation (Zod)
├── [feature].types.ts        # TypeScript types
└── __tests__/                # Tests
```

### 1. User Module (`modules/user/`)

**Purpose:** User profile management and onboarding.

**Key Endpoints:**

- `GET /api/v1/users/me` - Get current user
- `POST /api/v1/users/me/onboarding` - Complete onboarding
- `PUT /api/v1/users/me` - Update profile
- `DELETE /api/v1/users/me` - Soft delete account

**Key Features:**

- Soft delete (auto-restore on re-auth)
- Onboarding completion tracking
- Profile preferences

### 2. Sessions Module (`modules/sessions/`)

**Purpose:** Screenshot analysis, OCR, RAG, and AI suggestions.

**This is the most complex module!** See detailed docs:

- Sessions architecture docs:
  - [Upload Flow](./sessions/architecture/UPLOAD_FLOW.md)
  - [RAG Design](./sessions/architecture/RAG.md)
  - [WebSocket Flow](./sessions/architecture/WEBSOCKET.md)
- [Sessions API Contract](./sessions/SESSIONS_API_CONTRACT.md)

**Key Services:**

- `upload.service.ts` - R2 pre-signed URLs
- `ocr.service.ts` - Gemini Vision OCR
- `rag.service.ts` - Vector similarity search
- `embedding.service.ts` - OpenAI embeddings
- `phase-detection.service.ts` - Conversation phase analysis
- `suggestion.service.ts` - AI suggestion generation
- `cache.service.ts` - Redis caching
- `rate-limiter.service.ts` - Rate limiting
- `metrics.service.ts` - Cache metrics

**Key Endpoints:**

- `GET /api/v1/sessions/upload-url` - Get pre-signed upload URL
- `POST /api/v1/sessions` - Create session
- `GET /api/v1/sessions/:id` - Get session
- `POST /api/v1/sessions/:id/confirm-ocr` - Confirm OCR results
- `POST /api/v1/sessions/:id/generate-suggestions` - Generate AI suggestions
- `POST /api/v1/sessions/:id/messages` - Add message
- `PATCH /api/v1/sessions/:id/messages/:messageId` - Update message
- `DELETE /api/v1/sessions/:id/messages/:messageId` - Delete message

**Background Jobs:**

- `processSession.job.ts` - OCR processing (BullMQ)

### 3. Magic Opener Module (`modules/magic-opener/`)

**Purpose:** AI-generated conversation starters.

**Key Features:**

- Multi-provider AI (OpenAI, Anthropic, Gemini)
- Fallback strategy
- WebSocket streaming
- Cost tracking

**Key Endpoints:**

- `POST /api/v1/magic-openers/generate` - Generate opener
- `POST /api/v1/magic-openers/:id/toggle` - Like/dislike

**Documentation:** [Magic Openers Guide](./magic/MAGIC_OPENERS_BACKEND_GUIDE.md)

### 4. AI Module (`modules/ai/`)

**Purpose:** AI client abstraction with multi-provider support.

**Key Components:**

- `ai-client.ts` - Client factory
- `ai-strategy.ts` - Provider selection with fallbacks
- `providers/` - Provider implementations (OpenAI, Anthropic, Gemini)

**Strategy:**

1. Try primary provider (`AI_PROVIDER_STRATEGY_PRIMARY`)
2. Fallback to secondary providers if primary fails
3. Track costs per request

### 5. Webhook Module (`modules/webhook/`)

**Purpose:** Handle Clerk webhooks for user synchronization.

**Key Features:**

- User creation/update/delete sync
- Webhook signature verification (Svix)
- Idempotency handling

**Endpoint:** `POST /api/webhooks/clerk`

### 6. Analytics Module (`modules/analytics/`)

**Purpose:** Cost tracking for AI operations.

**Key Service:** `cost-tracker.service.ts`

**Tracks:**

- Token usage
- Cost per request
- Daily cost limits

### 7. Notification Module (`modules/notification/`)

**Purpose:** Push notifications via Expo.

**Key Service:** `notification.service.ts`

**Usage:**

```typescript
await notificationService.sendPushNotification({
  token: "ExponentPushToken[...]",
  title: "Daily Tip",
  body: "Your daily dating tip is ready!",
});
```

---

## 💻 Development Workflow

### Daily Development

**Recommended:** Use Neon development branch (`.env.local`)

```bash
# Start local server (connects to Neon dev branch)
npm run dev:local

# In another terminal, run tests
npm run test:watch
```

### Environment Setup

**Three Environment Modes:**

1. **Local Development (`.env.local`)** - RECOMMENDED

   - Uses Neon development branch
   - No Docker needed for database
   - Fast iteration

2. **Docker Development (`.env.docker`)**

   - Full stack (DB + Redis + API)
   - Good for integration testing
   - Requires Docker

3. **Testing (`.env.test`)**
   - Isolated test database
   - Automatic cleanup

### Code Quality

```bash
# Lint
npm run lint
npm run lint:fix

# Format
npm run format

# Type check
npm run type-check

# All checks
npm run lint && npm run type-check && npm test
```

### Git Workflow

1. **Create feature branch**

   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **Make changes**

   - Write code
   - Add tests
   - Update documentation

3. **Pre-commit checks** (automatic via Husky)

   - Lint
   - Format
   - Run related tests

4. **Pre-push checks** (automatic)

   - Type check
   - Full test suite

5. **Create PR**
   - CI/CD runs full checks
   - Code review
   - Merge

---

## 🧪 Testing

### Test Structure

```
__tests__/
├── unit/              # Unit tests (mocked dependencies)
├── integration/       # Integration tests (real DB)
└── __fixtures__/      # Test data
```

### Running Tests

```bash
# Setup test database (first time)
npm run test:setup-db

# Run all tests
npm test

# Run unit tests only
npm run test:unit

# Run integration tests only
npm run test:integration

# Watch mode
npm run test:watch

# Coverage
npm run test:coverage

# Cleanup
npm run test:teardown-db
```

### Writing Tests

**Unit Test Example:**

```typescript
describe("UserService", () => {
  it("should create user", async () => {
    const mockPrisma = createMockPrisma();
    const service = createUserService(mockPrisma);

    const user = await service.createUser({ email: "test@example.com" });

    expect(user.email).toBe("test@example.com");
    expect(mockPrisma.user.create).toHaveBeenCalled();
  });
});
```

**Integration Test Example:**

```typescript
describe("POST /api/v1/sessions", () => {
  it("should create session", async () => {
    const response = await request(app)
      .post("/api/v1/sessions")
      .set("Authorization", `Bearer ${token}`)
      .send({ type: "screenshot" });

    expect(response.status).toBe(201);
    expect(response.body.data.id).toBeDefined();
  });
});
```

### Test Database

- Uses isolated PostgreSQL container (`test-db`)
- Port: `5433` (separate from dev `5432`)
- Automatic schema application
- Clean state per test run

---

## 🗄️ Database & Migrations

### Prisma Schema

**Location:** `prisma/schema.prisma`

**Key Models:**

- `User` - User profiles
- `Session` - Chat sessions
- `Message` - Messages within sessions
- `KnowledgeChunk` - RAG knowledge base
- `AIGeneratedOpener` - Magic Openers

### Migrations

**Development (Neon dev branch):**

```bash
# Create migration
npm run db:migrate:create -- --name add_new_field

# Apply migration
npm run db:migrate

# Reset database (WARNING: deletes all data)
npm run db:reset
```

**Production:**

```bash
# Deploy migrations
npm run db:migrate:deploy
```

### Prisma Studio

```bash
# Open Prisma Studio (Neon dev branch)
npm run db:studio

# Opens at http://localhost:5555
```

### Common Operations

```bash
# Check migration status
npm run db:migrate:status

# Push schema without migration (dev only)
npm run db:push

# Pull schema from database
npm run db:pull
```

---

## 📡 API Documentation

### Base URL

- **Development:** `http://localhost:8080`
- **Production:** `https://api.charisma.ai`

### Authentication

All protected endpoints require Clerk JWT token:

```bash
Authorization: Bearer <clerk-jwt-token>
```

### Response Format

**Success:**

```json
{
  "success": true,
  "data": { ... }
}
```

**Error:**

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message"
  }
}
```

### Key Endpoints

See detailed API documentation:

- [Sessions API](./sessions/SESSIONS_API_CONTRACT.md)
- [Magic Openers API](./magic/MAGIC_OPENERS_BACKEND_GUIDE.md)

### Health Checks

```bash
# Basic health check
GET /health

# Queue health check
GET /health/queue
```

---

## 🚀 Deployment

For the full, canonical deployment guide, see [`DEPLOYMENT.md`](./DEPLOYMENT.md).

### Environment Setup

**Production uses Doppler for secrets:**

- Set `DOPPLER_TOKEN` in hosting platform
- Dockerfile runs: `doppler run -- node dist/server.js`

### Deployment Steps

1. **Build**

   ```bash
   npm run build
   ```

2. **Run migrations**

   ```bash
   npm run db:migrate:deploy
   ```

3. **Deploy**
   - Railway (current hosting)
   - Docker container
   - Environment variables via Doppler

### Infrastructure

- **Hosting:** Railway
- **Database:** Neon (PostgreSQL)
- **Cache/Queue:** Upstash Redis
- **Storage:** Cloudflare R2
- **CDN:** Cloudflare

---

## 🔍 Troubleshooting

### Common Issues

#### 1. Database Connection Errors

**Symptom:** `Can't reach database server`

**Solutions:**

- Check `.env.local` has correct `DATABASE_URL`
- Verify Neon development branch is active
- Check connection string format: `postgresql://user:pass@host/db?sslmode=require`

#### 2. Redis Connection Errors

**Symptom:** `Redis connection failed`

**Solutions:**

- Check `REDIS_URL` in environment
- Verify Upstash Redis is accessible
- For local dev, ensure Redis container is running: `docker compose up cache`

#### 3. Clerk Authentication Errors

**Symptom:** `401 Unauthorized`

**Solutions:**

- Verify `CLERK_SECRET_KEY` is correct
- Check token format in request headers
- Ensure token is not expired

#### 4. OCR Service Not Available

**Symptom:** `OCR service not available`

**Solutions:**

- Check `GEMINI_API_KEY` is set
- Verify API key is valid
- Check service initialization in `server.ts`

#### 5. Migration Errors

**Symptom:** `Migration failed`

**Solutions:**

- Check pgvector extension is enabled: `CREATE EXTENSION IF NOT EXISTS vector;`
- Verify database user has migration permissions
- Check migration files are in correct order

### Debugging Tips

1. **Check Logs**

   ```bash
   # Docker logs
   docker compose logs api

   # Local logs (in console)
   # Logs are printed to stdout
   ```

2. **Enable Debug Logging**

   ```bash
   LOG_LEVEL=debug npm run dev:local
   ```

3. **Database Inspection**

   ```bash
   npm run db:studio
   ```

4. **Test Individual Services**

   ```bash
   # Test OCR
   npm run demo-session-flow

   # Test notifications
   npm run test-notification -- <token>
   ```

---

## 📚 Next Steps

### Week 1: Foundation

1. ✅ Complete this onboarding guide
2. ✅ Set up local development environment
3. ✅ Run the application locally
4. ✅ Skim the Sessions architecture docs (UPLOAD_FLOW, RAG, WEBSOCKET)
5. ✅ Explore the codebase structure

### Week 2: Deep Dive

1. Read [Sessions API Contract](./sessions/SESSIONS_API_CONTRACT.md)
2. Understand RAG system (vector similarity search)
3. Study WebSocket implementation
4. Review test patterns
5. Make your first small change

### Week 3: Hands-On

1. Fix a small bug
2. Add a new endpoint
3. Write tests for your changes
4. Review PR process
5. Ask questions!

### Recommended Reading Order

1. **This document** (you're here!)
2. [Backend README](../../backend/README.md)
3. [Development Guide](../../backend/DEVELOPMENT.md)
4. Sessions architecture docs:
   - [Upload Flow](./sessions/architecture/UPLOAD_FLOW.md)
   - [RAG Design](./sessions/architecture/RAG.md)
   - [WebSocket Flow](./sessions/architecture/WEBSOCKET.md)
5. [Sessions API Contract](./sessions/SESSIONS_API_CONTRACT.md)
6. [Magic Openers Guide](./magic/MAGIC_OPENERS_BACKEND_GUIDE.md)
7. [Backend Tech Stack](./BACKEND_TECH_STACK_COMPLETE.md)

---

## 🆘 Getting Help

### Resources

- **Documentation:** `docs/backend/`
- **Code Comments:** Inline documentation in code
- **Tests:** Tests serve as usage examples
- **Team:** Ask questions in team chat

### Key Contacts

- **Tech Lead:** [Name]
- **Backend Team:** [Slack Channel]
- **Onboarding Buddy:** [Name]

---

## ✅ Onboarding Checklist

- [ ] Environment set up and running
- [ ] Can start development server
- [ ] Can run tests successfully
- [ ] Understand project structure
- [ ] Read Sessions architecture docs (UPLOAD_FLOW, RAG, WEBSOCKET)
- [ ] Read Sessions API Contract doc
- [ ] Made first code change
- [ ] Created first PR
- [ ] Understand deployment process

---

**Welcome to the team! 🎉**

If you have questions or need clarification, don't hesitate to ask. Good luck!
