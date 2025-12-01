## 1. Purpose and Scope

This document defines the **standard controller pattern** for the backend.
It is the single source of truth for how HTTP controllers and routes should be structured across all modules.

Objectives:

- Maximize readability and onboarding speed for new engineers.
- Make dependencies explicit and easy to reason about.
- Keep testing straightforward (no hidden factories or global state).

The chosen pattern is **Higher-Order Functions (HOF)**: each controller export is a function that takes its dependencies and returns an Express handler.

---

## 2. Rationale: Why Higher-Order Functions

We evaluated multiple approaches (factory objects, classes, direct handlers) and standardized on HOF controllers for the following reasons:

| Priority              | HOF Pattern (Standard)     | Factory / Class-Based Pattern        |
| --------------------- | -------------------------- | ------------------------------------ |
| Readability           | Immediate understanding    | Requires understanding factories     |
| Onboarding            | Simple in < 5 minutes      | Needs additional explanation         |
| Testing               | Direct, no global wiring   | Often needs factory setup / indirection |
| DRY (dependency wiring) | Slight repetition in routers | Centralized injection, less repetition |

We accept **minor repetition in routers** in exchange for:

- Explicit and local visibility of dependencies.
- No hidden “magic” in controller construction.
- Simpler mental model for both junior and senior engineers.

---

## 3. Standard Module Structure

Every module must follow this directory layout:

```text
src/modules/
└── [module-name]/
    ├── [module-name].controller.ts    # Higher-order handler functions
    ├── [module-name].routes.ts        # Route definitions and dependency wiring
    ├── [module-name].service.ts       # Business logic
    └── [module-name].types.ts         # Types used within the module
```

Notes:

- Controllers should not contain business logic; they are responsible for:
  - Reading `req` (params, body, headers, context).
  - Calling the service.
  - Mapping service results to HTTP responses.
- Services encapsulate business logic and should be reusable from jobs or other services.

---

## 4. Controller Pattern (Higher-Order Functions)

### 4.1 Core pattern

Each controller export:

- Is a function that accepts its dependencies (usually one or more services).
- Returns an Express-compatible handler `(req, res, next) => Promise<void>`.

```typescript
// [module-name].controller.ts
import type { Request, Response, NextFunction } from 'express';
import type { UserService } from './user.service.js';
import { success } from '../../core/http/response.js';

export const updateUserProfile =
  (userService: UserService) =>
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const result = await userService.updateProfile({
        userId: req.user!.id,
        payload: req.body,
      });

      success(res, { data: result });
    } catch (error) {
      next(error);
    }
  };
```

Key points:

- No classes, no `this`, no global singletons.
- All dependencies are passed in via parameters.
- The returned handler is a pure function of `(req, res, next)` plus its captured dependencies.

### 4.2 Router usage

Routers are responsible for:

- Instantiating services.
- Wiring dependencies into controllers.

```typescript
// [module-name].routes.ts
import { Router } from 'express';
import { updateUserProfile } from './user.controller.js';
import { createUserService } from './user.service.js';

export const createUserRoutes = () => {
  const router = Router();
  const userService = createUserService();

  router.put('/me', updateUserProfile(userService));

  return router;
};
```

The main app then mounts the routes:

```typescript
// app.ts
import { createUserRoutes } from './modules/user/user.routes.js';

app.use('/api/users', createUserRoutes());
```

---

## 5. Current Adoption Status

All active modules are expected to use this pattern:

| Module        | Status       | Key Dependencies                                                                                          |
| ------------- | ------------ | --------------------------------------------------------------------------------------------------------- |
| `user`        | Implemented  | `userService`                                                                                            |
| `magic-opener` | Implemented | `magicOpenerService`                                                                                     |
| `webhook`     | Implemented  | `webhookService`                                                                                         |
| `sessions`    | Implemented  | `sessionService`, `uploadService`, `reactionService`, `rateLimiterService`, `metricsService`, `queue?`  |

Any new module must follow the same HOF controller pattern and module structure.

---

## 6. Implementation Guidelines

### 6.1 Controller template

Use the following template when adding a new handler:

```typescript
// Standard export format
export const handlerName =
  (service: ServiceType /*, other deps */) =>
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const result = await service.doSomething(/* input from req */);

      success(res, { data: result });
    } catch (error) {
      next(error);
    }
  };
```

Rules:

- Always wrap logic in `try/catch` and call `next(error)` on failure.
- Use shared response helpers (`success`, `created`, etc.).
- Do not reach into global singletons from within handlers; always pass dependencies explicitly.

### 6.2 Routes factory

Routes should be created via a factory function that wires all dependencies:

```typescript
export interface ModuleRoutesDependencies {
  service: ServiceType;
  // other dependencies, e.g. rateLimiterService, metricsService, queue, etc.
}

export const createModuleRoutes = (deps: ModuleRoutesDependencies) => {
  const router = Router();

  router.get('/endpoint', handlerName(deps.service));

  return router;
};
```

Benefits:

- Clear DI boundary for each module.
- Easy to test route factories by passing mocked services if needed.

### 6.3 Dependency injection in the app

```typescript
// In main app setup
import { createModuleRoutes } from './modules/module/module.routes.js';
import { createModuleService } from './modules/module/module.service.js';

const moduleService = createModuleService();

app.use(
  '/api/module',
  createModuleRoutes({
    service: moduleService,
    // other dependencies
  })
);
```

---

## 7. Testing Controllers

One of the main advantages of HOF controllers is test simplicity.

```typescript
// [module-name].controller.test.ts
import { handlerName } from './module.controller.js';

test('handlerName sends expected response', async () => {
  const mockService = {
    doSomething: jest.fn().mockResolvedValue({ ok: true }),
  };

  const handler = handlerName(mockService as any);

  const req = mockRequest(/* ... */);
  const res = mockResponse();
  const next = jest.fn();

  await handler(req as any, res as any, next);

  expect(mockService.doSomething).toHaveBeenCalled();
  expect(res.status).toHaveBeenCalledWith(200);
  expect(res.json).toHaveBeenCalledWith(
    expect.objectContaining({ data: { ok: true } })
  );
  expect(next).not.toHaveBeenCalled();
});
```

Guidelines:

- Test controllers in isolation by mocking services.
- For complex flows, prefer testing the service directly and keep controller tests thin.

---

## 8. Variations and Special Cases

### 8.1 Handlers without external services

When the handler only reads from `req` and does not need a service:

```typescript
export const getCurrentUser =
  () =>
  async (req: Request, res: Response): Promise<void> => {
    // `req.user` is assumed to be populated by authentication middleware
    success(res, { user: req.user });
  };

// Usage:
// router.get('/me', loadUser, getCurrentUser());
```

Even in this case, keep the HOF shape `() => handler` for consistency.

### 8.2 Handlers with multiple dependencies

Some handlers require multiple services/queues:

```typescript
export const createSession =
  (
    sessionService: SessionService,
    uploadService: UploadService,
    queue?: SessionsQueue
  ) =>
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const session = await sessionService.create(/* ... */);

      if (queue) {
        await queue.enqueue(session.id);
      }

      created(res, { data: session });
    } catch (error) {
      next(error);
    }
  };
```

Keep dependencies as explicit parameters; do not group them into opaque “context” objects unless there is a strong reason.

---

## 9. Checklist for New Modules

When creating a new module, follow this checklist:

- [ ] Create `[module].service.ts` with business logic.
- [ ] Create `[module].controller.ts` with HOF-based handlers.
- [ ] Create `[module].routes.ts` that:
  - Instantiates services.
  - Wires handlers with explicit dependencies.
  - Applies shared middleware (for example, `loadUser`) at the router level.
- [ ] Use validation middleware (for example, `createValidationMiddleware`) for inputs.
- [ ] Use shared response helpers (`success`, `created`, etc.).
- [ ] Export individual handler functions from the controller (no controller classes or factories).

---

## 10. FAQ and Common Pitfalls

### 10.1 “Why am I repeating dependencies in the router?”

This is intentional.
Seeing `handler(service)` at the route definition makes dependencies explicit and easy to understand.
The small amount of repetition is an acceptable trade-off for clarity.

### 10.2 “Should I create a controller factory object for many handlers?”

No.
Export individual HOF handlers instead of a single controller factory.
If a module becomes too large, split it into smaller modules or sub-controllers.

### 10.3 “How do I handle shared middleware?”

Apply middleware at the router level:

```typescript
router.use(authMiddleware);
router.use(loadUser);

router.get('/me', getCurrentUser());
```

Do not bake shared middleware into individual handlers.

---

## 11. References

- `user.controller.ts` and `user.routes.ts` — basic examples.
- `sessions.controller.ts` — complex, multi-dependency examples.
- All modules in `backend/src/modules/` are expected to follow this pattern.
- For architectural rationale and alternatives, see `ARCHITECTURE_ANALYSIS.md` (rationale only; not a normative standard).
