## Sessions Backend Feature – Onboarding Guide

### 1. Overview

**Goal:** This guide helps backend engineers understand, extend, and safely work on the **Sessions** feature.  
**Scope:** Architecture, key flows, main services, routes, and how to add or change behavior following existing backend standards.

The Sessions feature is responsible for:

- Uploading screenshots or text conversations.
- OCR + RAG + phase detection + AI suggestions.
- Session/message CRUD, reactions, and metrics.
- Rate limiting and caching around AI-heavy flows.

---

### 2. Mental Model

At a high level, a **Session** is a conversation context for the user. The main pipeline is:

1. Upload – user uploads a screenshot or starts a text-based session.
2. OCR – a background job extracts text from screenshots.
3. Messages – user and AI messages are stored and updated.
4. RAG + phase detection – system understands context and conversation phase.
5. Suggestions – AI generates tailored suggestions.
6. Reactions and metrics – user feedback and cache/latency metrics are tracked.

Core architectural patterns:

- Higher-order controllers – handlers are higher-order functions that receive dependencies.
- Service factories – pure business logic, no `class` syntax.
- Route factories – inject dependencies once and wire middleware + handlers.

For controller pattern details, see `docs/backend/CONTROLLER_PATTERN_STANDARD.md`.

---

### 3. Key Files and Structure

Location: `backend/src/modules/sessions/`

- `sessions.controller.ts` – Express handlers (Higher-Order Functions)
- `sessions.routes.ts` – Route factory that wires middleware + handlers
- `sessions.service.ts` – Core Session domain logic (CRUD, RAG, suggestions)
- `sessions.validation.ts` – Zod schemas for body/query/params
- `sessions.types.ts` – DTOs and internal types for services
- `services/`
  - `upload.service.ts` – Pre-signed URLs + storage utilities
  - `ocr.service.ts` – OCR flow for screenshots
  - `embedding.service.ts` – OpenAI embeddings for RAG
  - `rag.service.ts` – Vector search and retrieval logic
  - `suggestion.service.ts` – AI suggestion generation
  - `phase-detection.service.ts` (+ `phase-detection/*`) – Phase detection engine
  - `cache.service.ts` – Caching around suggestions/phase results
  - `rate-limiter.service.ts` – Redis + in-memory rate limiting
  - `metrics.service.ts` – Cache and latency metrics
  - `reaction.service.ts` – Message reactions
  - `ingestion.service.ts` – Knowledge base ingestion
- `jobs/`
  - `queue.ts`, `worker.ts`, `processSession.job.ts` – BullMQ queue and workers
- `prompts/`
  - OCR and suggestions prompt templates
- `__tests__/` – Unit + integration tests for all of the above

For low-level internals (especially phase detection), see:

- `docs/backend/sessions/PHASE_DETECTION_SYSTEM.md`

---

### 4. Architecture Overview

#### 4.1 Controllers (higher-order functions)

- Located in `sessions.controller.ts`
- Each handler is a higher-order function receiving its dependencies and returning `(req, res, next) => Promise<void>`
- Example (simplified):

```typescript
export const getUploadUrl = (uploadService: UploadService) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    // ...
  };
};
```

Best practices:

- Do not access Prisma or low-level services directly in controllers.
- Call service methods and orchestrate high-level flow + HTTP responses.
- Use `success()` / `created()` helpers and `AppError` for errors.

#### 4.2 Routes (factory)

- Located in `sessions.routes.ts`.
- `createSessionRoutes(config: SessionRoutesConfig)` builds a `Router` with:
  - `clerkAuth` + `loadUser` middleware
  - Per-endpoint validation via `createValidationMiddleware`
  - Rate limiting via `createRateLimitMiddleware`
  - Controllers wired with injected services

```typescript
export const createSessionRoutes = ({
  sessionService,
  uploadService,
  reactionService,
  rateLimiterService,
  metricsService,
  sessionsQueue,
}: SessionRoutesConfig) => {
  const router = Router();

  router.use(clerkAuth);
  router.use(loadUser);

  router.post(
    "/",
    createValidationMiddleware(createSessionBodySchema, "body"),
    createRateLimitMiddleware(rateLimiterService, "general"),
    createSession(sessionService, uploadService, sessionsQueue)
  );

  // ...other routes...

  return router;
};
```

Best practices:

- Add new endpoints in the module route file, not in global routes.
- Always add validation and rate limiting where applicable.
- Keep route handlers small by delegating to services.

#### 4.3 Service layer (factory functions)

- Core entry point: `createSessionService(deps: SessionServiceDependencies)`.
- Responsibilities:
  - Session & message CRUD
  - RAG queries
  - Orchestrating Phase Detection + Suggestions
  - Integrating cache and metrics where necessary

Example (simplified):

```typescript
export const createSessionService = ({
  prisma,
  ragService,
  phaseDetectionService,
  suggestionService,
  cacheService,
}: SessionServiceDependencies) => {
  const createSession = async (input: CreateSessionInput): Promise<Session> => {
    // Prisma create + mapping
  };

  const listSessions = async (
    userId: string,
    filters: SessionFilters
  ): Promise<SessionListResult> => {
    // Prisma query + filters + mapping
  };

  // ...other methods (messages, suggestions, etc.)...

  return {
    createSession,
    listSessions,
    // ...
  };
};
```

Best practices:

- Keep services framework-agnostic (no `Request`/`Response`).
- Map Prisma models to DTOs defined in `sessions.types.ts`.
- Log with `createServiceLogger('sessions')` at important steps.
- Use `AppError` for domain/validation errors that bubble up to HTTP.

---

### 5. Core Flows

#### 5.1 Creating a session (screenshot or text)

Entry point:

- `POST /api/v1/sessions`
- Handled by `createSession` controller + `sessionService.createSession`.

High-level steps:

1. Validate body (`createSessionBodySchema`).
2. Apply rate limiting via `rateLimiterService`.
3. Create a `Session` record via Prisma.
4. If the type is `screenshot`:
   - Extract `imageKey` from URL via `uploadService`.
   - Enqueue `processSession` job via `SessionsQueue`.
5. Return `201 Created` with the created session DTO.

Where to look:

- `sessions.controller.ts` → `createSession`.
- `sessions.service.ts` → `createSession`.
- `jobs/processSession.job.ts` → downstream OCR + phase/RAG flow.

#### 5.2 Upload URL and OCR flow

Upload URL:

- `GET /api/v1/sessions/upload-url`.
- Uses `uploadService.generatePresignedUrl` to return a pre-signed Cloudflare R2 URL.

OCR:

- Triggered by `processSession.job.ts` (BullMQ worker) after a screenshot session is created.
- Uses `ocr.service.ts` + prompt from `prompts/ocr-prompt.ts`.
- Updates session status and stores OCR result in the database.

#### 5.3 Listing and viewing sessions

- `GET /api/v1/sessions` → `listSessions` with filters and pagination.
- `GET /api/v1/sessions/:id` → `getSession` with full message history.

Best practices:

- Only return active sessions (`deletedAt = null`).
- Use `SessionFilters` types instead of ad-hoc query logic in controllers.

#### 5.4 Messages and reactions

- `POST /api/v1/sessions/:id/messages` – create a message (user/contact/AI).
- `PATCH /api/v1/sessions/:id/messages/:messageId` – update a message.
- `DELETE /api/v1/sessions/:id/messages/:messageId` – soft delete a message.
- `POST /api/v1/sessions/:id/messages/:messageId/reactions` – like/dislike/pin, etc.

These route into `SessionService` and `ReactionService`:

- Maintain ordered message history.
- Track likes/dislikes/pins and usage counts.
- Update `lastActivity` and related fields as needed.

#### 5.5 AI suggestions

- `POST /api/v1/sessions/:id/generate-suggestions`.
- Request validation:
  - Params: session ID.
  - Body: `GenerateSuggestionsBody` (filters such as `aiStyle`, `customTwist`, `aiLanguage`).

Flow:

1. Load session + full message history.
2. Run RAG via `rag.service.ts` to fetch relevant knowledge chunks.
3. Run phase detection to determine conversation phase and allowed next phase.
4. Use `suggestion.service.ts` with retrieved chunks, phase info, and filters.
5. Optionally cache results via `cache.service.ts` and track metrics via `metrics.service.ts`.

Where to look:

- `sessions.service.ts` – main suggestion orchestration.
- `services/phase-detection.*` – phase-detection logic.
- `services/suggestion.service.ts` + `prompts/*` – AI calls and prompt construction.

#### 5.6 Metrics, cache, and rate limiting

- `GET /api/v1/sessions/metrics/cache` – internal cache metrics endpoint.
- `rate-limiter.service.ts` – central rate limiter (used via `createRateLimitMiddleware`).
- `metrics.service.ts` – tracks cache hits/misses and latencies.
- `cache.service.ts` – wraps caching around suggestions and phase detection.

Guidelines:

- All AI-heavy endpoints must be rate-limited.
- Any new cache should expose metrics (hit ratio, latency).

---

### 6. How to Add a New Endpoint

Example: add `GET /api/v1/sessions/:id/summary`.

1. Define types and validation:
   - Add DTO to `sessions.types.ts` (for example, `SessionSummary`).
   - Add a Zod schema to `sessions.validation.ts` if needed.
2. Extend service:
   - Add `getSessionSummary(sessionId, userId)` to `createSessionService`.
3. Expose controller:
   - Add `export const getSessionSummary = (sessionService: SessionService) => ...` to `sessions.controller.ts`.
4. Wire route:
   - In `sessions.routes.ts`, add:

```typescript
router.get(
  "/:id/summary",
  createValidationMiddleware(sessionIdParamsSchema, "params"),
  createRateLimitMiddleware(rateLimiterService, "general"),
  getSessionSummary(sessionService)
);
```

5. Tests:
   - Unit test for `getSessionSummary` in `__tests__/unit/sessions.service.test.ts`.
   - Integration test in `__tests__/integration/sessions.api.test.ts`.

---

### 7. Testing Strategy

Tests live under `backend/src/modules/sessions/__tests__/`:

- **Unit tests:**
  - Services: `sessions.service.test.ts`, `rag.service.test.ts`, `phase-detection.service.test.ts`, `suggestion.service.test.ts`, etc.
  - Low-level helpers (phase rules, complexity scores, cache, rate limiter, metrics, upload, ocr)
- **Integration tests:**
  - `sessions.api.test.ts` – REST API flows
  - `suggestions-flow.integration.test.ts`, `filters-flow.integration.test.ts`, etc.
  - `websocket.test.ts` for streaming behavior

**Commands (examples):**

```bash
# All tests
npm test

# Focus on sessions module
npm test -- sessions

# Focus on phase detection
npm test -- phase-detection
```

Best practices:

- For any new behavior, add at least:
  - One unit test at the service level.
  - One integration test if it affects external behavior (API or WebSocket).
- Prefer pure functions in services to simplify testing.

---

### 8. Suggested Onboarding Path

1. Read (skim):
   - `docs/backend/CONTROLLER_PATTERN_STANDARD.md` – controller pattern standard.
   - This onboarding guide (current file).
2. Browse code:
   - `sessions.routes.ts` – see all endpoints and middleware.
   - `sessions.controller.ts` – map handlers to service calls.
   - `sessions.service.ts` – understand key domain methods.
3. Run tests for safety:

```bash
npm test -- sessions
```

4. Make a small change:
   - Example: extend filters for `listSessions` or adjust metrics.
   - Add or update tests to cover your change.

When in doubt, follow the existing patterns in the Sessions module and mirror how similar behavior is already implemented elsewhere. The goal is to keep everything predictable, testable, and explicit.
