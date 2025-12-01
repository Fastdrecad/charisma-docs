## Backend Tech Stack – Complete Guide

## 1. Overview

This document describes the backend tech stack, patterns, conventions, and best practices.
It is the source of truth for the backend stack and should generally be read after `ONBOARDING.md` by any new backend developer (especially those working on the Sessions feature).

---

## 2. Runtime & Framework

### Runtime

**Node.js 22+ (Alpine)**

- **Version:** Node.js 22 (visible in `Dockerfile`)
- **Base Image:** `node:22-alpine`
- **ESM Modules:** `"type": "module"` in `package.json`
- **Module System:** Native ESM (`.js` extensions in imports)

**Location:**

- `backend/Dockerfile:2`
- `backend/package.json:5`

---

### Framework

**Express.js 5.1.0**

- **Version:** `^5.1.0` (latest major version)
- **Type:** RESTful API framework
- **Middleware:** Helmet, CORS, Pino HTTP logging
- **Body Parser:** `express.json({ limit: '10mb' })`

**Location:**

- `backend/package.json:70`
- `backend/src/app.ts`

**Why Express?**

- ✅ Mature and stable library
- ✅ Large ecosystem
- ✅ Good TypeScript support
- ✅ Easy testing

---

### Language

**TypeScript (Strict Mode)**

- **Version:** `^5.9.3`
- **Strict Mode:** ✅ Enabled
- **Module:** `nodenext` (native ESM)
- **Target:** `esnext`

**Strict Options:**

```json
{
  "strict": true,
  "noUncheckedIndexedAccess": true,
  "exactOptionalPropertyTypes": true,
  "verbatimModuleSyntax": true,
  "isolatedModules": true
}
```

**Location:**

- `backend/package.json:106`
- `backend/tsconfig.json`

---

## 3. Database & ORM

### Database

**PostgreSQL (Neon)**

- **Provider:** Neon (serverless PostgreSQL)
- **Connection:** `DATABASE_URL` environment variable
- **Migrations:** Prisma Migrate
- **Connection Pooling:** Neon built-in pooling

**Location:**

- `backend/prisma/schema.prisma:5-8`
- `backend/src/core/env/index.ts:13`

---

### ORM

**Prisma 7.0.0**

- **Version:** `7.0.0`
- **Type:** Type-safe ORM
- **Migrations:** ✅ Prisma Migrate
- **Client Generation:** `prisma generate`

**Migrations:**

```bash
# Development
npm run db:migrate              # Create and apply migration
npm run db:migrate:create       # Create migration only
npm run db:migrate:status       # Check migration status

# Production
npm run db:migrate:deploy       # Apply migrations (no prompt)
```

**Location:**

- `backend/package.json:19-28`
- `backend/prisma/schema.prisma`

**Why Prisma?**

- ✅ Type-safe queries
- ✅ Auto-generated TypeScript types
- ✅ Migrations built-in
- ✅ Excellent DX (Developer Experience)

---

### Redis

**Redis 7 (Alpine)**

- **Version:** `redis:7-alpine` (Docker)
- **Usage:** Rate limiting, caching, pub/sub
- **Connection:** `REDIS_URL` environment variable
- **Required:** ✅ Yes (required env var)

**Location:**

- `backend/docker-compose.yml:50-56`
- `backend/src/core/env/index.ts:16`

**Usage:**

- Rate limiting (WebSocket, REST API)
- Pub/Sub for real-time updates (Sessions status/phase updates, job progress)
- BullMQ queue connection for background jobs
- Caching (optional)

---

## 4. File Storage & Media Processing

### File Storage

**Status: ❌ NOT IMPLEMENTED**

**Planned:**

- **Storage:** S3 or Cloudflare R2 (pre-signed URLs)
- **Pattern:** Pre-signed URLs for direct upload
- **Flow:** `GET /presigned-url` → Client uploads directly → `POST /sessions` with URL

**Architecture (from docs):**

```
Client → GET /presigned-url → Backend → S3/R2
Client → Upload directly to S3/R2
Client → POST /sessions (with imageUrl) → Backend
```

**Location (design docs):**

- `docs/backend/magic/MAGIC_OPENERS_BACKEND_GUIDE.md`

**For Sessions Feature:**

- Need to implement pre-signed URL endpoint
- Need to choose storage provider (S3/R2)
- Need to add image processing (resize, optimization)

---

### Image Processing

**Status: ❌ NOT IMPLEMENTED**

**Recommended:**

- **Library:** Sharp (fastest, best for Node.js)
- **Alternative:** Jimp (pure JavaScript, slower)

**For Sessions Feature:**

- Resize screenshots (max width/height)
- Optimize file size (compression)
- Generate thumbnails (optional)

---

### CDN

**Status: ❌ NOT IMPLEMENTED**

**Recommended:**

- Cloudflare CDN (if using R2)
- AWS CloudFront (if using S3)
- Direct S3/R2 serving (if sufficient)

---

## 5. AI/ML Integration

### LLM Providers

**Multi-Provider Strategy with Fallback**

**Primary Provider:**

- **Default:** `gpt-4o` (OpenAI)
- **Configurable:** `AI_PROVIDER_STRATEGY_PRIMARY` env var
- **Options:** `gemini-flash`, `gpt-4o`, `claude-sonnet`

**Fallback Providers:**

- **Default:** `claude-sonnet` (Anthropic)
- **Configurable:** `AI_PROVIDER_STRATEGY_FALLBACKS` env var
- **Format:** Comma-separated list

**Supported Providers:**

1. **Google Gemini Flash** (`@google/genai`)
2. **OpenAI GPT-4o** (`openai`)
3. **Anthropic Claude Sonnet** (`@anthropic-ai/sdk`)

**Location:**

- `backend/src/core/env/index.ts:39-52`
- `backend/src/modules/ai/ai-strategy.ts` (assumed)

**Strategy Pattern:**

```typescript
const aiStrategy = createAiStrategy(aiClient, {
  primary: env.AI_PROVIDER_STRATEGY_PRIMARY,
  fallbacks: env.AI_PROVIDER_STRATEGY_FALLBACKS,
});
```

---

### OCR Service

**Status: ✅ IMPLEMENTED**

**Current Implementation (Sessions Feature):**

- **Provider:** Google Gemini Vision via `@google/genai`
- **Service:** `OCRService` with `processScreenshot(imageUrl)` API
- **Prompting:** Uses a structured `OCR_PROMPT` for extracting messages and metadata
- **Timeout & Errors:** Wrapped with explicit timeout (`GEMINI_TIMEOUT_MS`) and `AppError` codes from `error.catalog.ts`

**Location:**

- `backend/src/modules/sessions/services/ocr.service.ts`
- `backend/src/modules/sessions/prompts/ocr-prompt.ts`

---

### Existing AI Integration

**Magic Openers Feature:**

- ✅ AI streaming (real-time generation)
- ✅ Multi-provider support
- ✅ Fallback mechanism
- ✅ Cost tracking

**Location:**

- `backend/src/modules/magic-opener/magic-opener.service.ts:27-102`

---

## 6. Real-time & Background Jobs

### WebSocket

**Native `ws` Library**

- **Library:** `ws` (native WebSocket)
- **Version:** (in dependencies, but not explicitly stated)
- **Pattern:** Post-upgrade authentication
- **Heartbeat:** JSON ping/pong (React Native compatibility)

**Features:**

- ✅ Post-upgrade authentication (JWT token in message)
- ✅ Rate limiting (per-user, per-IP)
- ✅ Abort signal support (cancellation)
- ✅ Heartbeat (ping/pong)

**Location:**

- `backend/src/core/websocket/websocket-server.ts`

**For Sessions Feature:**

- Can use the same WebSocket server
- Add `subscribe` action for job updates
- Redis pub/sub for real-time notifications

---

### Background Jobs

**Status: ✅ IMPLEMENTED (Sessions Feature)**

**Current Implementation:**

- **Queue:** BullMQ (Redis-backed) – `Queue<ProcessSessionJobPayload>`
- **Pattern:** Job queue with WebSocket notifications
- **Flow:**
  ```
  POST /sessions → Enqueue job in "sessions" queue
  Worker (BullMQ) processes job → updates DB / publishes progress
  WebSocket → Sends status/ocr/suggestion updates to client
  ```

**Location:**

- `backend/src/modules/sessions/jobs/queue.ts`
- `backend/src/modules/sessions/jobs/worker.ts`

---

### Worker System

**Status: ✅ IMPLEMENTED (Sessions Worker)**

**Current Implementation:**

- **BullMQ Worker:** `createSessionsWorker` processes jobs from the `"sessions"` queue
- **Concurrency:** Configured concurrency (e.g. `concurrency: 5`)
- **Error Handling:** Logs queue/worker errors via shared logger

**Location:**

- `backend/src/modules/sessions/jobs/worker.ts`

---

## 7. Authentication & Authorization

### Auth System

**Clerk (JWT Tokens)**

- **Provider:** Clerk
- **Type:** JWT tokens
- **Library:** `@clerk/backend`
- **Verification:** `verifyToken()` function

**Flow:**

```
Client → Clerk login → JWT token
Client → API request → Authorization: Bearer <token>
Backend → verifyToken(token) → Clerk ID
Backend → loadUser(Clerk ID) → Internal DB User ID
```

**Location:**

- `backend/src/core/env/index.ts:55-56`
- `backend/src/modules/auth/auth.middleware.ts`

---

### User Model Structure

**Dual ID System:**

- **Clerk ID:** External auth ID (`clerkId`)
- **Internal ID:** Database ID (`id` - CUID)

**Schema:**

```prisma
model User {
  id          String   @id @default(cuid())
  clerkId     String   @unique @map("clerk_id")
  email       String   @unique

  // Profile fields
  firstName   String?
  lastName    String?
  // ... other fields
}
```

**Mapping:**

- `clerkAuth` middleware → Verifies JWT, extracts Clerk ID
- `loadUser` middleware → Maps Clerk ID → Internal DB User ID
- `req.user.id` → Internal DB User ID (used in handlers)

**Location:**

- `backend/prisma/schema.prisma:10-55`
- `backend/src/modules/auth/loadUser.middleware.ts`

---

### Permission System

**Status: ❌ NOT IMPLEMENTED**

**Current:**

- ✅ User ownership check (user can access only their data)
- ❌ Role-based access control (RBAC)
- ❌ Permission system

**For Sessions Feature:**

- User ownership is sufficient (user sees only their sessions)
- No need for RBAC at this moment

---

## 8. API Architecture

### API Style

**RESTful API**

- **Style:** RESTful
- **Versioning:** URL-based (`/api/v1/...`)
- **Response Format:** JSON
- **Status Codes:** Standard HTTP status codes

**Response Structure:**

```typescript
// Success
{
  success: true,
  data: { ... }
}

// Error
{
  success: false,
  error: {
    code: string,
    message: string,
    details?: unknown
  }
}
```

**Location:**

- `backend/src/app.ts:113-117`
- `backend/src/core/http/response.ts`

---

### API Versioning

**URL-based Versioning**

- **Pattern:** `/api/v1/...`
- **Current Version:** `v1`
- **Future:** `/api/v2/...` for breaking changes

**Location:**

- `backend/src/app.ts:113-117`

---

### API Documentation

**Status: ❌ NOT IMPLEMENTED**

**Recommended:**

- **OpenAPI/Swagger:** For API documentation
- **Library:** `swagger-jsdoc` + `swagger-ui-express`
- **Alternative:** Manual documentation (current)

**For Sessions Feature:**

- Recommended adding OpenAPI documentation
- Or at least a detailed README with endpoints

---

## 9. Testing & Code Quality

### Testing Framework

**Jest 29.7.0**

- **Version:** `^29.7.0`
- **Preset:** `ts-jest/presets/default-esm`
- **Environment:** Node.js
- **Coverage:** Jest coverage reports

**Test Types:**

- **Unit Tests:** `*.test.ts` (service, validation, completion)
- **Integration Tests:** `*.integration.test.ts`

**Scripts:**

```bash
npm run test              # All tests
npm run test:unit         # Unit tests only
npm run test:integration  # Integration tests only
npm run test:coverage     # Coverage report
npm run test:watch        # Watch mode
```

**Location:**

- `backend/package.json:38-44`
- `backend/jest.config.cjs`

---

### E2E Testing

**Status: ❌ NOT IMPLEMENTED**

**Recommended:**

- **Supertest:** For API endpoint testing
- **Library:** `supertest` (already in dependencies)
- **Pattern:** Integration tests with real database

**Location:**

- `backend/package.json:102` (supertest dependency)

---

### Linting & Formatting

**ESLint + Prettier**

**ESLint:**

- **Version:** `^9.38.0`
- **Config:** `@typescript-eslint/recommended`, `prettier`
- **Parser:** `@typescript-eslint/parser`
- **Rules:** Strict TypeScript rules

**Prettier:**

- **Version:** `^3.6.2`
- **Integration:** ESLint Prettier plugin

**Scripts:**

```bash
npm run lint        # Check linting
npm run lint:fix   # Fix linting issues
npm run format      # Format code
```

**Location:**

- `backend/.eslintrc.json`
- `backend/package.json:32-34`

**Pre-commit Hooks:**

- **Husky:** Pre-commit hooks
- **lint-staged:** Format only staged files
- **Auto-fix:** ESLint + Prettier on commit

**Location:**

- `backend/package.json:108-113`

---

## 10. Deployment & Infrastructure

### Hosting

**Status: ⚠️ NOT EXPLICITLY STATED**

**From Dockerfile:**

- **Base:** `node:22-alpine`
- **Doppler:** Secrets management (Doppler CLI)
- **Multi-stage build:** Optimized for production

**Recommended:**

- **Railway:** Easy deployment, good integration
- **Render:** Alternative
- **AWS/GCP/Azure:** For enterprise
- **Docker:** Container-based deployment

**Location:**

- `backend/Dockerfile`

---

### Container Orchestration

**Docker**

- **Dockerfile:** Multi-stage build
- **Docker Compose:** For local development
- **Services:** PostgreSQL, Redis

**Docker Compose:**

```yaml
services:
  db: # PostgreSQL
  cache: # Redis
  api: # Backend API (optional)
```

**Location:**

- `backend/Dockerfile`
- `backend/docker-compose.yml`

**Kubernetes:**

- ❌ Not implemented
- ⚠️ Might be needed for production scaling

---

### CI/CD

**Status: ⚠️ NOT EXPLICITLY STATED IN BACKEND**

**From frontend documentation:**

- **GitHub Actions:** Automated testing
- **Pre-commit Hooks:** Lint + Format
- **Pre-push Hooks:** Type-check + Tests

**Recommended for Backend:**

- GitHub Actions for CI/CD
- Automated testing before deployment
- Docker build and push
- Automated migrations

---

### Monitoring & Logging

**Logging: Pino**

- **Library:** `pino` + `pino-http`
- **Format:** JSON (production), Pretty (development)
- **Levels:** fatal, error, warn, info, debug, trace
- **Structured Logging:** ✅ Yes

**Configuration:**

```typescript
logger.info({ userId, streamId }, "Starting generation");
logger.error({ error, userId }, "Generation failed");
logger.debug({ provider }, "AI provider selected");
```

**Location:**

- `backend/src/core/logger/index.ts`
- `backend/src/app.ts:48-68`

**Monitoring:**

- ❌ Sentry: Not implemented
- ❌ DataDog: Not implemented
- ❌ LogRocket: Not implemented

**Recommended:**

- **Sentry:** Error tracking
- **DataDog/New Relic:** APM (Application Performance Monitoring)
- **Pino:** Structured logging (already implemented)

---

## 11. Existing Patterns & Conventions

### Folder Structure

```
backend/
├── src/
│   ├── app.ts                    # Express app factory
│   ├── server.ts                 # Server entry point
│   ├── config/                   # Configuration
│   ├── core/                     # Core utilities
│   │   ├── ai/                   # AI client & strategy
│   │   ├── database/             # Prisma client
│   │   ├── env/                  # Environment variables
│   │   ├── error/                # Error handling
│   │   ├── http/                 # HTTP utilities
│   │   ├── logger/               # Logging
│   │   ├── middleware/           # Middleware
│   │   └── websocket/            # WebSocket server
│   ├── modules/                  # Feature modules
│   │   ├── magic-opener/         # Magic Openers feature
│   │   │   ├── magic-opener.service.ts
│   │   │   ├── magic-opener.controller.ts
│   │   │   ├── magic-opener.routes.ts
│   │   │   ├── magic-opener.types.ts
│   │   │   ├── magic-opener.validation.ts
│   │   │   └── prompts/          # AI prompts
│   │   ├── user/                 # User feature
│   │   └── webhook/              # Webhook feature
│   ├── routes/                   # Global routes
│   └── shared/                   # Shared utilities
├── prisma/
│   └── schema.prisma             # Database schema
└── package.json
```

**Conventions:**

- ✅ Feature-based organization (`modules/`)
- ✅ Separation of concerns (service, controller, routes)
- ✅ Core utilities in `core/`
- ✅ Shared code in `shared/`

---

### Naming Conventions

**Files:**

- **Services:** `*.service.ts`
- **Controllers:** `*.controller.ts`
- **Routes:** `*.routes.ts`
- **Types:** `*.types.ts`
- **Validation:** `*.validation.ts`
- **Tests:** `*.test.ts`, `*.integration.test.ts`

**Functions:**

- **Factory Functions:** `create*` (e.g., `createMagicOpenerService`)
- **Handlers:** Verb-based (e.g., `getOpeners`, `likeOpener`)
- **Private Functions:** `_*` prefix (e.g., `_buildPrompt`)

**Variables:**

- **camelCase:** For variables and functions
- **PascalCase:** For types and classes
- **UPPER_CASE:** For constants
- **`_` prefix:** For intentionally unused variables

---

### Error Handling Patterns

**AppError Class:**

```typescript
export class AppError extends Error {
  constructor(
    public code: string,
    public message: string,
    public statusCode: number = 500,
    public details?: unknown
  ) {
    super(message);
  }
}
```

**Error Handler Middleware:**

```typescript
export const errorHandler = (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      success: false,
      error: {
        code: err.code,
        message: err.message,
        details: err.details,
      },
    });
  }

  // Unknown error
  logger.error({ err, req }, "Unhandled error");
  res.status(500).json({
    success: false,
    error: {
      code: "INTERNAL_ERROR",
      message: "An unexpected error occurred",
    },
  });
};
```

**Location:**

- `backend/src/core/error/AppError.ts`
- `backend/src/core/error/errorHandler.ts`

---

### Validation Library

**Zod 4.1.12**

- **Version:** `^4.1.12`
- **Usage:** Request validation, environment validation
- **Pattern:** Schema-first validation

**Request Validation:**

```typescript
export const interactionParamsSchema = z.object({
  openerId: z.cuid("Invalid opener ID format"),
});

// Middleware
router.post(
  "/:openerId/like",
  createValidationMiddleware(interactionParamsSchema, "params"),
  likeOpener(magicOpenerService)
);
```

**Environment Validation:**

```typescript
const envSchema = z.object({
  DATABASE_URL: z.string().min(1, "DATABASE_URL is required"),
  CLERK_SECRET_KEY: z.string().min(1, "CLERK_SECRET_KEY is required"),
  // ...
});

export const env = envSchema.parse(process.env);
```

**Location:**

- `backend/package.json:78`
- `backend/src/modules/magic-opener/magic-opener.validation.ts`
- `backend/src/core/env/index.ts`

---

## 12. Summary – Key Points for Sessions Feature

### 1. Use Existing Patterns

- ✅ **Service Layer:** Factory functions with dependency injection
- ✅ **Controller Layer:** Higher-order functions
- ✅ **Routes:** Factory functions
- ✅ **Validation:** Zod schemas
- ✅ **Error Handling:** AppError class

### 2. Implement Missing Parts

- ❌ **File Storage:** Pre-signed URLs (S3/R2)
- ❌ **Image Processing:** Sharp library
- ❌ **OCR Service:** Google Cloud Vision API
- ❌ **Background Jobs:** BullMQ + Redis
- ❌ **Worker System:** Separate process for job processing

### 3. Reuse Existing

- ✅ **WebSocket Server:** Same server, add `subscribe` action
- ✅ **AI Integration:** Same AI strategy pattern
- ✅ **Database:** Prisma with migrations
- ✅ **Authentication:** Clerk JWT tokens
- ✅ **Logging:** Pino structured logging

### 4. Tech Stack Checklist

- [x] Node.js 22+ (Alpine)
- [x] Express.js 5.1.0
- [x] TypeScript (Strict Mode)
- [x] PostgreSQL (Neon)
- [x] Prisma 6.18.0
- [x] Redis 7
- [ ] File Storage (S3/R2) - **NEEDS IMPLEMENTATION**
- [ ] Image Processing (Sharp) - **NEEDS IMPLEMENTATION**
- [ ] OCR Service (Google Vision) - **NEEDS IMPLEMENTATION**
- [x] AI Integration (Multi-provider)
- [x] WebSocket (`ws` library)
- [ ] Background Jobs (BullMQ) - **NEEDS IMPLEMENTATION**
- [x] Clerk Authentication
- [x] Zod Validation
- [x] Jest Testing
- [x] ESLint + Prettier

---

## 13. Code References

### Main Files

- **Package:** `backend/package.json`
- **TypeScript Config:** `backend/tsconfig.json`
- **Dockerfile:** `backend/Dockerfile`
- **Docker Compose:** `backend/docker-compose.yml`
- **ESLint Config:** `backend/.eslintrc.json`
- **Jest Config:** `backend/jest.config.cjs`

### Core Modules

- **App Factory:** `backend/src/app.ts`
- **Server Entry:** `backend/src/server.ts`
- **Environment:** `backend/src/core/env/index.ts`
- **Logger:** `backend/src/core/logger/index.ts`
- **Error Handler:** `backend/src/core/error/errorHandler.ts`
- **WebSocket:** `backend/src/core/websocket/websocket-server.ts`

### Feature Modules

- **Magic Opener Service:** `backend/src/modules/magic-opener/magic-opener.service.ts`
- **Magic Opener Controller:** `backend/src/modules/magic-opener/magic-opener.controller.ts`
- **Magic Opener Routes:** `backend/src/modules/magic-opener/magic-opener.routes.ts`

### Database

- **Schema:** `backend/prisma/schema.prisma`

---

**End of Documentation**
