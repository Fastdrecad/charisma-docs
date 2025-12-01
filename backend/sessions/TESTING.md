## Sessions API – Testing Guide

**Status:** Active  
**Last Updated:** 2025-01-XX  
**Purpose:** Comprehensive testing guide for the Sessions feature (unit, integration, E2E, load, and manual testing).

---

### 1. Overview

This document describes a complete testing strategy for the Sessions API:

1. Unit tests – individual function and service tests.
2. Integration tests – end-to-end API flow tests.
3. E2E tests – Postman collection with complete scenarios.
4. Load testing – performance and rate limiting tests.
5. Manual testing – QA testing checklist.

---

### 2. Unit Tests

### Location

```
backend/src/modules/sessions/__tests__/unit/
```

### How to run

```bash
cd backend
npm run test:unit
```

### Coverage

- ✅ `sessions.service.test.ts` - Service layer logic
- ✅ `sessions.controller.test.ts` - Controller handlers
- ✅ `rag.service.test.ts` - RAG retrieval
- ✅ `phase-detection.service.test.ts` - Phase detection
- ✅ `suggestion.service.test.ts` - AI suggestion generation
- ✅ `cache.service.test.ts` - Caching logic
- ✅ `rate-limiter.service.test.ts` - Rate limiting
- ✅ `metrics.service.test.ts` - Metrics tracking
- ✅ `upload.service.test.ts` - Upload service

### Example test

```typescript
describe("SessionService", () => {
  it("should create session with correct status", async () => {
    const session = await sessionService.createSession({
      userId: "user_123",
      type: "screenshot",
    });

    expect(session.status).toBe("pending_user_input");
    expect(session.type).toBe("screenshot");
  });
});
```

---

---

### 3. Integration Tests

### Location

```
backend/src/modules/sessions/__tests__/integration/
```

### How to run

```bash
cd backend
npm run test:integration
```

### Test scenarios

#### 2.1 Sessions API flow

**File:** `sessions.api.test.ts`

- ✅ Create session (screenshot & text)
- ✅ Get session by ID
- ✅ List sessions (pagination, filters)
- ✅ Delete session (soft delete)
- ✅ Rate limit headers

#### 2.2 OCR flow

**File:** `sessions.integration.test.ts`

- ✅ Create session with screenshot
- ✅ Confirm OCR results
- ✅ OCR results converted to messages
- ✅ Session status transitions

#### 2.3 Message CRUD flow

**File:** `message-crud-flow.integration.test.ts`

- ✅ Add message (user, contact, AI)
- ✅ Update message
- ✅ Delete message
- ✅ Cache invalidation

#### 2.4 Suggestions flow

**File:** `suggestions-flow.integration.test.ts`

- ✅ Generate suggestion
- ✅ Phase detection
- ✅ RAG retrieval
- ✅ Filter support (aiStyle, customTwist, aiLanguage)

#### 2.5 Filters flow

**File:** `filters-flow.integration.test.ts`

- ✅ Filter by status
- ✅ Filter by type
- ✅ Pagination
- ✅ Sorting

---

---

### 4. Postman Collection (E2E Testing)

### Installation

1. **Import Collection:**

   ```
   File → Import → Sessions_API.postman_collection.json
   ```

2. **Import Environment:**

   ```
   File → Import → Sessions_API.postman_environment.json
   ```

3. **Set Clerk token:**
   - Open environment variables
   - Set `clerkToken` with a valid Clerk JWT token

### How to run

#### Manual testing

1. Open Postman
2. Select environment: "Sessions API - Local"
3. Run requests in order (1–8)

#### Automated testing (Newman)

```bash
# Install Newman
npm install -g newman

# Run collection
newman run backend/postman/Sessions_API.postman_collection.json \
  -e backend/postman/Sessions_API.postman_environment.json \
  --env-var "clerkToken=your-token-here"

# Run with HTML report
newman run backend/postman/Sessions_API.postman_collection.json \
  -e backend/postman/Sessions_API.postman_environment.json \
  --env-var "clerkToken=your-token-here" \
  -r html \
  --reporter-html-export reports/postman-report.html
```

### Collection structure

#### 1. Authentication

- Get Clerk Token (Manual)

#### 2. Upload flow

- ✅ Get Upload URL
- ✅ Get Upload URL - With Size Validation
- ✅ Get Upload URL - Image Too Large (11MB)

#### 3. Session CRUD

- ✅ Create Session - Screenshot
- ✅ Create Session - Text
- ✅ Get Session by ID
- ✅ List Sessions
- ✅ List Sessions - Filter by Status
- ✅ Delete Session

#### 4. OCR flow

- ✅ Confirm OCR

#### 5. Messages CRUD

- ✅ Add Message - User
- ✅ Add Message - Contact
- ✅ Update Message
- ✅ Delete Message

#### 6. AI suggestions

- ✅ Generate Suggestion
- ✅ Generate Suggestion - With Custom Twist

#### 7. Rate limiting tests

- ✅ Check Rate Limit Headers
- ✅ Test Rate Limit Exceeded

#### 8. Error handling tests

- ✅ Invalid Session ID
- ✅ Invalid Content Type

---

---

### 5. Load Testing

### Tool: Artillery.io

**Installation:**

```bash
npm install -g artillery
```

**Load test config:**

```yaml
# backend/artillery/sessions-load-test.yml
config:
  target: "http://localhost:8080"
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Sustained load"
    - duration: 60
      arrivalRate: 100
      name: "Spike test"
  defaults:
    headers:
      Authorization: "Bearer {{ $processEnvironment.CLERK_TOKEN }}"

scenarios:
  - name: "Create and List Sessions"
    flow:
      - post:
          url: "/api/v1/sessions"
          json:
            type: "text"
            name: "Load Test Session"
      - get:
          url: "/api/v1/sessions"
```

**How to run:**

```bash
artillery run backend/artillery/sessions-load-test.yml
```

### Tool: k6

**Script:**

```javascript
// backend/k6/sessions-load-test.js
import http from "k6/http";
import { check } from "k6";

export const options = {
  stages: [
    { duration: "30s", target: 20 },
    { duration: "1m", target: 50 },
    { duration: "30s", target: 0 },
  ],
};

const BASE_URL = "http://localhost:8080";
const CLERK_TOKEN = __ENV.CLERK_TOKEN;

export default function () {
  // Create session
  const createRes = http.post(
    `${BASE_URL}/api/v1/sessions`,
    JSON.stringify({ type: "text", name: "Load Test" }),
    {
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${CLERK_TOKEN}`,
      },
    }
  );

  check(createRes, {
    "create session status 201": (r) => r.status === 201,
    "rate limit headers present": (r) =>
      r.headers["X-RateLimit-Limit"] !== undefined,
  });

  // List sessions
  const listRes = http.get(`${BASE_URL}/api/v1/sessions`, {
    headers: {
      Authorization: `Bearer ${CLERK_TOKEN}`,
    },
  });

  check(listRes, {
    "list sessions status 200": (r) => r.status === 200,
  });
}
```

**How to run:**

```bash
k6 run backend/k6/sessions-load-test.js --env CLERK_TOKEN=your-token
```

---

---

### 6. Manual Testing Checklist

### Pre-deployment testing

#### Authentication

- [ ] Valid Clerk token works
- [ ] Invalid token returns 401
- [ ] Missing token returns 401

#### Upload flow

- [ ] Get upload URL with valid contentType
- [ ] Get upload URL with invalid contentType (returns 400)
- [ ] Get upload URL with valid size (< 10MB)
- [ ] Get upload URL with invalid size (> 10MB, returns 400)
- [ ] Upload URL expires after 15 minutes

#### Session CRUD

- [ ] Create screenshot session
- [ ] Create text session
- [ ] Get session by ID
- [ ] List sessions (pagination)
- [ ] Filter by status
- [ ] Filter by type
- [ ] Delete session (soft delete)
- [ ] Restore deleted session

#### OCR flow

- [ ] Create session with screenshot
- [ ] Wait for OCR processing (WebSocket updates)
- [ ] Confirm OCR results
- [ ] OCR results converted to messages
- [ ] Session status → complete

#### Messages CRUD

- [ ] Add user message
- [ ] Add contact message
- [ ] Add AI message
- [ ] Update message text
- [ ] Delete message
- [ ] Cache invalidation works

#### AI suggestions

- [ ] Generate suggestion (default)
- [ ] Generate suggestion with aiStyle filter
- [ ] Generate suggestion with customTwist
- [ ] Generate suggestion with aiLanguage
- [ ] Phase detection works correctly
- [ ] RAG retrieval works
- [ ] Suggestion persisted as AI message

#### Rate limiting

- [ ] Rate limit headers present on all endpoints
- [ ] Rate limit exceeded returns 429
- [ ] Rate limit resets after window
- [ ] Different limits for different endpoints

#### Error handling

- [ ] Invalid session ID returns 404
- [ ] Invalid request body returns 400
- [ ] Missing required fields returns 400
- [ ] Server errors return 500 with error code

---

---

### 7. Test Coverage Goals

### Target coverage

- **Unit Tests:** > 80%
- **Integration Tests:** > 70%
- **E2E Tests:** 100% of critical paths

### Current coverage

```bash
npm run test:coverage
```

**Expected:**

```
Statements   : 85%+
Branches     : 80%+
Functions    : 85%+
Lines        : 85%+
```

---

---

### 8. CI/CD Integration

### GitHub Actions example

```yaml
# .github/workflows/test-sessions.yml
name: Test Sessions API

on:
  push:
    paths:
      - "backend/src/modules/sessions/**"
  pull_request:
    paths:
      - "backend/src/modules/sessions/**"

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: "20"

      - name: Install dependencies
        run: |
          cd backend
          npm ci

      - name: Run unit tests
        run: |
          cd backend
          npm run test:unit

      - name: Run integration tests
        run: |
          cd backend
          npm run test:integration
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
          REDIS_URL: ${{ secrets.TEST_REDIS_URL }}

      - name: Run Postman tests
        run: |
          npm install -g newman
          newman run backend/postman/Sessions_API.postman_collection.json \
            -e backend/postman/Sessions_API.postman_environment.json \
            --env-var "clerkToken=${{ secrets.TEST_CLERK_TOKEN }}"
```

---

---

### 9. Test Scenarios Documentation

### Happy path

1. User creates screenshot session
2. Uploads image to R2
3. OCR processes image
4. User confirms OCR results
5. Messages are created
6. User generates AI suggestion
7. User adds new message
8. User generates another suggestion

### Error scenarios

1. Invalid authentication token
2. Invalid session ID
3. Image too large (> 10MB)
4. Rate limit exceeded
5. Invalid request body
6. Session not in correct status

### Edge cases

1. Empty OCR results
2. Very long message text
3. Multiple rapid requests
4. Concurrent session creation
5. Deleted session access

---

---

### 10. Performance Benchmarks

### Target metrics

| Endpoint                   | Target p95 | Target p99 |
| -------------------------- | ---------- | ---------- |
| GET /sessions              | < 100ms    | < 200ms    |
| POST /sessions             | < 200ms    | < 500ms    |
| POST /generate-suggestions | < 2s       | < 3s       |
| GET /sessions/:id          | < 50ms     | < 100ms    |

### Monitoring

- Use APM tools (New Relic, Datadog)
- Track response times
- Monitor error rates
- Track rate limit hits

---

---

### 11. References

- **API Contract:** `docs/backend/sessions/SESSIONS_API_CONTRACT.md`
- **Architecture:** `docs/backend/sessions/ARCHITECTURE.md`
- **Postman Collection:** `backend/postman/Sessions_API.postman_collection.json`
- **Integration Tests:** `backend/src/modules/sessions/__tests__/integration/`

---

---

### 12. Troubleshooting

### Common Issues

#### Database Connection Errors

```bash
# Check database is running
docker-compose ps

# Reset test database
npm run test:db:reset
```

#### Redis Connection Errors

```bash
# Check Redis is running
redis-cli ping

# Reset Redis
redis-cli FLUSHALL
```

#### Rate Limit Issues

```bash
# Reset rate limits (development only)
POST /api/v1/sessions/dev/reset-rate-limit
{
  "type": "general"
}
```

---

**Last Updated:** 2025-01-XX  
**Maintained by:** Backend Team
