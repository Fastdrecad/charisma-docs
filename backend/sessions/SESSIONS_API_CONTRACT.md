## Sessions API Contract

**Purpose:** Authoritative API specification for frontend–backend integration of the Sessions feature.  
**Frontend reference:** `client/src/schemas/session.schema.ts`

---

### 1. REST API Endpoints

#### 1.1 Endpoint summary

| Endpoint                                                | Method | Purpose                                 |
| ------------------------------------------------------- | ------ | --------------------------------------- |
| GET /api/v1/sessions/upload-url                         | GET    | Pre-signed URL (R2)                     |
| POST /api/v1/sessions                                   | POST   | Create session                          |
| GET /api/v1/sessions/:id                                | GET    | Get session                             |
| GET /api/v1/sessions                                    | GET    | List sessions                           |
| DELETE /api/v1/sessions/:id                             | DELETE | Delete session (soft or hard)           |
| PATCH /api/v1/sessions/:id/restore                      | PATCH  | Restore soft-deleted session            |
| PATCH /api/v1/sessions/:id                              | PATCH  | Update session metadata (name/pin)      |
| POST /api/v1/sessions/:id/confirm-ocr                   | POST   | Confirm OCR results                     |
| POST /api/v1/sessions/:id/retry-ocr                     | POST   | Retry OCR processing                    |
| POST /api/v1/sessions/:id/generate-suggestions          | POST   | AI suggestions                          |
| POST /api/v1/sessions/:id/messages                      | POST   | Add message                             |
| PATCH /api/v1/sessions/:id/messages/:messageId          | PATCH  | Edit message                            |
| DELETE /api/v1/sessions/:id/messages/:messageId         | DELETE | Delete message                          |
| POST /api/v1/sessions/:id/messages/:messageId/reactions | POST   | Add / toggle reaction (like, pin, etc.) |
| GET /api/v1/sessions/metrics/cache                      | GET    | Cache metrics (internal/admin)          |
| POST /api/v1/sessions/dev/reset-rate-limit              | POST   | Reset rate limit (development)          |

---

### 2. Detailed Endpoint Specifications

#### 2.1 GET `/api/v1/sessions/upload-url`

**Purpose:** Get pre-signed URL for direct R2 upload

**Request:**

```typescript
// Query params
{
  contentType: "image/jpeg" | "image/png" | "image/webp";
}
```

**Response (200 OK):**

```json
{
  "uploadUrl": "https://r2-presigned-url...",
  "imageKey": "sessions/user-123/screenshot-456.jpg",
  "expiresAt": 1234567890 // Unix timestamp (seconds)
}
```

**Errors:**

- `400 Bad Request`: Invalid contentType
- `401 Unauthorized`: Missing/invalid auth token
- `500 Internal Server Error`: R2 service error

---

#### 2.2 POST `/api/v1/sessions`

**Purpose:** Create new session

**Request:**

```json
{
  "type": "screenshot" | "text",
  "name": "Optional session name",
  "imageUrl": "https://r2-url/sessions/user-123/screenshot-456.jpg"  // Required for screenshot type
}
```

**Response (201 Created):**

```json
{
  "session": {
    "id": "session_123",
    "type": "screenshot",
    "status": "uploading", // Immediately queues job
    "name": "Optional session name",
    "screenshotUrl": "https://r2-url/...",
    "ocrResult": null,
    "createdAt": 1234567890000, // Unix timestamp (ms)
    "updatedAt": 1234567890000,
    "lastActivity": 1234567890000,
    "isActive": true,
    "isPinned": false,
    "messages": [],
    "messageCount": 0
  }
}
```

**Errors:**

- `400 Bad Request`: Invalid request body, missing imageUrl for screenshot type
- `401 Unauthorized`: Missing/invalid auth token
- `500 Internal Server Error`: Database error

---

#### 2.3 GET `/api/v1/sessions/:id`

**Purpose:** Get single session by ID

**Response (200 OK):**

```json
{
  "session": {
    "id": "session_123",
    "type": "screenshot",
    "status": "verifying_ocr",
    "name": "Optional session name",
    "screenshotUrl": "https://r2-url/...",
    "ocrResult": [
      { "id": "ocr_1", "text": "Hi", "author": "contact" },
      { "id": "ocr_2", "text": "Hello!", "author": "user" }
    ],
    "createdAt": 1234567890000,
    "updatedAt": 1234567890000,
    "lastActivity": 1234567890000,
    "isActive": true,
    "isPinned": false,
    "messages": [],
    "messageCount": 0
  }
}
```

**Errors:**

- `401 Unauthorized`: Missing/invalid auth token
- `403 Forbidden`: Session belongs to different user
- `404 Not Found`: Session not found
- `500 Internal Server Error`: Database error

---

#### 2.4 GET `/api/v1/sessions`

**Purpose:** List user's sessions

**Query Params:**

```typescript
{
  limit?: number;      // Default: 50, Max: 100
  offset?: number;     // Default: 0
  status?: SessionStatus;  // Filter by status
  type?: "screenshot" | "text";  // Filter by type
}
```

**Response (200 OK):**

```json
{
  "sessions": [
    {
      "id": "session_123",
      "type": "screenshot",
      "status": "complete",
      "name": "Session name",
      "screenshotUrl": "https://r2-url/...",
      "ocrResult": null,
      "createdAt": 1234567890000,
      "updatedAt": 1234567890000,
      "lastActivity": 1234567890000,
      "isActive": true,
      "isPinned": false,
      "messages": [],
      "messageCount": 0
    }
  ],
  "total": 100,
  "limit": 50,
  "offset": 0
}
```

**Errors:**

- `401 Unauthorized`: Missing/invalid auth token
- `400 Bad Request`: Invalid query params
- `500 Internal Server Error`: Database error

---

#### 2.5 DELETE `/api/v1/sessions/:id`

**Purpose:** Delete session. Supports soft delete by default, with optional permanent deletion via query param.

**Request:**

```text
DELETE /api/v1/sessions/:id?permanent=false
```

- `permanent` (optional, default: `false`):
  - `false` → soft delete (`deletedAt` set, session hidden from lists)
  - `true` → hard delete (permanent removal)

**Response (204 No Content):**

```text
(empty body)
```

**Errors:**

- `401 Unauthorized`: Missing/invalid auth token
- `403 Forbidden`: Session belongs to different user
- `404 Not Found`: Session not found
- `500 Internal Server Error`: Database error

---

#### 2.6 PATCH `/api/v1/sessions/:id/restore`

**Purpose:** Restore a previously soft-deleted session.

**Response (200 OK):**

```json
{
  "session": {
    "id": "session_123",
    "status": "pending_user_input",
    "isActive": true,
    "deletedAt": null,
    "...": "other session fields"
  }
}
```

**Errors:**

- `401 Unauthorized`: Missing/invalid auth token
- `403 Forbidden`: Session belongs to different user
- `404 Not Found`: Session not found or not deleted
- `500 Internal Server Error`: Database error

---

#### 2.7 PATCH `/api/v1/sessions/:id`

**Purpose:** Update session metadata (currently: `name`, `isPinned`).

**Request:**

```json
{
  "name": "New session name", // Optional
  "isPinned": true // Optional
}
```

At least one field must be provided; otherwise, the request is rejected.

**Response (200 OK):**

```json
{
  "session": {
    "id": "session_123",
    "name": "New session name",
    "isPinned": true,
    "...": "other session fields"
  }
}
```

**Errors:**

- `400 Bad Request`: No updatable fields provided
- `401 Unauthorized`: Missing/invalid auth token
- `403 Forbidden`: Session belongs to different user
- `404 Not Found`: Session not found
- `500 Internal Server Error`: Database error

---

#### 2.8 POST `/api/v1/sessions/:id/confirm-ocr`

**Purpose:** Confirm OCR results and convert to messages. Accepts `ocrResult` in request body (allows user to confirm modified OCR results, e.g., after swapping senders).

**Request:**

```json
{
  "ocrResult": [
    { "id": "ocr_1", "text": "Hi", "author": "contact" },
    { "id": "ocr_2", "text": "Hello!", "author": "user" }
  ]
}
```

**Response (200 OK):**

```json
{
  "session": {
    "id": "session_123",
    "status": "complete",
    "ocrResult": null,  // Removed after confirmation
    "messages": [
      { "id": "msg_1", "text": "Hi", "author": "contact", "timestamp": 1234567890000, ... },
      { "id": "msg_2", "text": "Hello!", "author": "user", "timestamp": 1234567891000, ... }
    ],
    "messageCount": 2,
    ...
  }
}
```

**Errors:**

- `400 Bad Request`: Invalid `ocrResult` format, missing or empty array
- `401 Unauthorized`: Missing/invalid auth token
- `403 Forbidden`: Session belongs to different user
- `404 Not Found`: Session not found
- `409 Conflict`: Session already confirmed (status is not 'verifying_ocr')
- `500 Internal Error`: Database error

**Notes:**

- Session must be in `verifying_ocr` status
- `ocrResult` must be provided in request body (non-empty array)
- Frontend can send modified OCR results (e.g., after user swaps senders)
- After confirmation, `ocrResult` in session is cleared (set to `null`)
- Messages are created with staggered timestamps (1 second apart)
- Operation is atomic (transaction-based)

---

#### 2.9 POST `/api/v1/sessions/:id/generate-suggestions`

**Purpose:** Generate AI suggestions based on chat history

**Request:**

```json
{
  "aiStyle": "witty", // Optional, from user profile
  "customTwist": "..." // Optional
}
```

**Response (200 OK):**

```json
{
  "suggestion": {
    "id": "suggestion_1",
    "text": "AI generated suggestion text",
    "title": "Custom title",
    "tag": "Attraction",
    "analysis": "Why this works explanation"
  }
}
```

**Note:** Returns a single AI suggestion (not an array). For multiple suggestions, make multiple requests.

**Errors:**

- `401 Unauthorized`: Missing/invalid auth token
- `403 Forbidden`: Session belongs to different user
- `404 Not Found`: Session not found
- `400 Bad Request`: Session has no messages (can't generate suggestions)
- `500 Internal Server Error`: AI service error

---

#### 2.10 POST `/api/v1/sessions/:id/messages`

**Purpose:** Add new message to session

**Request:**

```json
{
  "text": "Message text",
  "author": "user" | "contact" | "ai",
  "timestamp": 1234567890000  // Optional, defaults to now
}
```

**Response (201 Created):**

```json
{
  "message": {
    "id": "msg_123",
    "text": "Message text",
    "author": "user",
    "timestamp": 1234567890000,
    "sentiment": null,
    "collectionIds": [],
    "isLiked": false, // Calculated from UserMessageInteraction (LIKED)
    "isDisliked": false, // Calculated from UserMessageInteraction (DISLIKED)
    "isPinned": false, // Calculated from UserMessageInteraction (PINNED)
    "usedCount": 0, // Calculated from UserMessageInteraction (COPIED_TO_CLIPBOARD count)
    "title": null,
    "tag": null,
    "analysis": null
  }
}
```

**Note:** `isLiked`, `isDisliked`, `isPinned`, and `usedCount` are calculated from `UserMessageInteraction` table (consistent with Magic Openers pattern). These fields are computed in the service layer based on the current user's interactions.

**Errors:**

- `400 Bad Request`: Invalid request body
- `401 Unauthorized`: Missing/invalid auth token
- `403 Forbidden`: Session belongs to different user
- `404 Not Found`: Session not found
- `409 Conflict`: Session not in 'complete' status
- `500 Internal Server Error`: Database error

---

## 🔄 Iterative Flow (Conversation Building)

The Sessions feature supports an **iterative flow** where the user can:

1. **Get an AI suggestion** → `POST /api/v1/sessions/:id/generate-suggestions`
2. **Add a new message** → `POST /api/v1/sessions/:id/messages`
3. **Request a new suggestion** → `POST /api/v1/sessions/:id/generate-suggestions` (with expanded history)

### Example Flow:

**Step 1: Initial OCR (2 messages)**

```json
messages: [
  { "author": "contact", "text": "Hi" },
  { "author": "user", "text": "Hello!" }
]
```

**Step 2: Generate suggestion**

```
POST /api/v1/sessions/123/generate-suggestions

→ Returns 1 AI suggestion
```

**Step 3: User selects/adds a new message**

```
POST /api/v1/sessions/123/messages

{
  "author": "user",
  "text": "How was your day?"
}
```

**Step 4: Contact replies (user adds message)**

```
POST /api/v1/sessions/123/messages

{
  "author": "contact",
  "text": "Pretty good, went to the gym"
}
```

**Step 5: Generate new suggestion (with 4-message history)**

```
POST /api/v1/sessions/123/generate-suggestions

→ RAG uses ALL 4 messages as context
→ Suggestion is more relevant (more context)
```

### Key Points:

- ✅ `generateSuggestions` **always** uses ALL messages from the session
- ✅ The RAG query is built from the **full history** (not only OCR messages)
- ✅ Context **expands** with each new request
- ✅ There is no limit on the number of iterations
- ✅ Each request returns **1 suggestion** (for multiple suggestions, make multiple requests)

### Implementation Details:

```typescript
// generateSuggestions always uses ALL messages
const generateSuggestions = async (sessionId, filters) => {
  // 1. Get ALL messages from the session (not just initial OCR)
  const session = await prisma.session.findUnique({
    where: { id: sessionId },
    include: {
      messages: {
        orderBy: { timestamp: "asc" }, // Chronological order
      },
    },
  });

  // 2. Build conversation context from ALL messages
  const conversationContext = session.messages
    .map((m) => `${m.author}: ${m.text}`)
    .join("\n");

  // 3. RAG retrieval with full context
  const relevantChunks = await ragService.retrieve(conversationContext, {
    topK: 5,
    filters: {
      phase: detectConversationPhase(session.messages.length), // Dynamic phase
    },
  });

  // 4. Generate suggestion with full context + RAG
  const suggestion = await aiProvider.generateSuggestion({
    conversationContext,
    ragChunks: relevantChunks,
    filters,
  });

  return suggestion; // Single suggestion, not array
};
```

#### 2.11 PATCH `/api/v1/sessions/:id/messages/:messageId`

**Purpose:** Edit message (e.g., update title)

**Request:**

```json
{
  "title": "New title" // Optional, can update other fields
}
```

**Response (200 OK):**

```json
{
  "message": {
    "id": "msg_123",
    "text": "Message text",
    "title": "New title",
    ...
  }
}
```

**Errors:**

- `400 Bad Request`: Invalid request body
- `401 Unauthorized`: Missing/invalid auth token
- `403 Forbidden`: Session/message belongs to different user
- `404 Not Found`: Session or message not found
- `500 Internal Server Error`: Database error

#### 2.12 DELETE `/api/v1/sessions/:id/messages/:messageId`

**Purpose:** Delete message from session.

**Response (204 No Content):**

```text
(empty body)
```

**Errors:**

- `401 Unauthorized`: Missing/invalid auth token
- `403 Forbidden`: Session/message belongs to different user
- `404 Not Found`: Session or message not found
- `500 Internal Server Error`: Database error

---

#### 2.13 POST `/api/v1/sessions/:id/messages/:messageId/reactions`

**Purpose:** Add or toggle a reaction on a message (like/dislike/pin/etc.), mirroring `UserMessageInteraction` semantics.

**Request:**

```json
{
  "type": "LIKED" | "DISLIKED" | "PINNED" | "COPIED_TO_CLIPBOARD"
}
```

**Response (200 OK):**

```json
{
  "message": {
    "id": "msg_123",
    "isLiked": true,
    "isDisliked": false,
    "isPinned": false,
    "usedCount": 3,
    "...": "other message fields"
  }
}
```

**Errors:**

- `400 Bad Request`: Invalid reaction type
- `401 Unauthorized`: Missing/invalid auth token
- `403 Forbidden`: Session/message belongs to different user
- `404 Not Found`: Session or message not found
- `500 Internal Server Error`: Database error

---

### 3. WebSocket Protocol (Sessions)

#### 3.1 Connection

**URL:** `wss://api.charismaai.app/ws` (production)  
**Authentication:** Via WebSocket message (post-upgrade), not headers/query params.

#### 3.2 Client → server messages

##### 3.2.1 Authenticate

```json
{
  "action": "authenticate",
  "payload": { "token": "clerk_jwt_token" }
}
```

**Response:**

```json
{
  "type": "authenticated",
  "message": "Authentication successful"
}
```

##### 3.2.2 Subscribe to session updates

```json
{
  "action": "sessions:subscribe",
  "payload": {
    "sessionId": "session_123"
  }
}
```

**Response:**

```json
{
  "type": "subscribed",
  "sessionId": "session_123",
  "message": "Subscribed to session updates"
}
```

##### 3.2.3 Unsubscribe from session updates

```json
{
  "action": "sessions:unsubscribe",
  "payload": {
    "sessionId": "session_123"
  }
}
```

**Response:**

```json
{
  "type": "unsubscribed",
  "sessionId": "session_123",
  "message": "Unsubscribed from session updates"
}
```

#### 3.3 Server → client messages

##### 3.3.1 Session status update

```json
{
  "type": "session_status",
  "sessionId": "session_123",
  "status": "analysis_in_progress",
  "message": "Analyzing screenshot...",
  "progress": 0.5 // Optional, 0.0 - 1.0
}
```

**Status values:**

- `uploading`
- `analysis_in_progress`
- `verifying_ocr`
- `complete`
- `error`

##### 3.3.2 Upload progress

```json
{
  "type": "session_upload_progress",
  "sessionId": "session_123",
  "progress": 0.75 // 0.0 - 1.0
}
```

##### 3.3.3 OCR complete

```json
{
  "type": "session_status",
  "sessionId": "session_123",
  "status": "verifying_ocr",
  "ocrResult": [
    { "id": "ocr_1", "text": "Hi", "author": "contact" },
    { "id": "ocr_2", "text": "Hello!", "author": "user" }
  ],
  "screenshotUrl": "https://r2-url/sessions/..."
}
```

##### 3.3.4 Error

```json
{
  "type": "session_error",
  "sessionId": "session_123",
  "status": "error",
  "error": {
    "code": "OCR_FAILED",
    "message": "Failed to analyze screenshot"
  }
}
```

---

### 4. Status Enum

**Frontend:** `client/src/schemas/session.schema.ts:8-16`

```typescript
type SessionStatus =
  | "pending_user_input"
  | "previewing_screenshot"
  | "uploading"
  | "analysis_in_progress"
  | "verifying_ocr"
  | "complete"
  | "error";
```

**Backend Prisma:**

```prisma
enum SessionStatus {
  PENDING_USER_INPUT
  PREVIEWING_SCREENSHOT
  UPLOADING
  ANALYSIS_IN_PROGRESS
  VERIFYING_OCR
  COMPLETE
  ERROR
}
```

**Mapping:** Convert in service layer (snake_case ↔ UPPER_SNAKE_CASE)

---

### 5. Status Transitions

```
pending_user_input
  ↓ (user selects image)
previewing_screenshot
  ↓ (user confirms)
uploading
  ↓ (upload complete, job queued)
analysis_in_progress
  ↓ (OCR complete)
verifying_ocr
  ↓ (user confirms OCR)
complete
  ↓ (error occurs)
error
```

**Allowed transitions:**

- `pending_user_input` → `previewing_screenshot`, `error`
- `previewing_screenshot` → `pending_user_input`, `uploading`, `error`
- `uploading` → `analysis_in_progress`, `error`
- `analysis_in_progress` → `verifying_ocr`, `error`
- `verifying_ocr` → `complete`, `error`
- `complete` → `error`
- `error` → `pending_user_input`, `complete`

---

### 6. Error Codes

| Code                  | HTTP Status | Description                       |
| --------------------- | ----------- | --------------------------------- |
| `OCR_FAILED`          | 500         | OCR service failed                |
| `UPLOAD_FAILED`       | 500         | Upload to R2 failed               |
| `INVALID_IMAGE`       | 400         | Image format not supported        |
| `IMAGE_TOO_LARGE`     | 400         | Image exceeds size limit (10MB)   |
| `RATE_LIMIT_EXCEEDED` | 429         | Too many requests                 |
| `SESSION_NOT_FOUND`   | 404         | Session ID not found              |
| `UNAUTHORIZED`        | 401         | Authentication required           |
| `FORBIDDEN`           | 403         | Session belongs to different user |
| `INVALID_REQUEST`     | 400         | Invalid request body              |

---

### 7. Timeout Values

- **Pre-signed URL expiry:** 15 minutes (900 seconds)
- **WebSocket reconnect:** 3 attempts, exponential backoff (1s, 2s, 4s)
- **Job timeout:** 60s (max processing time for OCR)
- **Upload timeout:** 30s (max time for R2 upload)
- **API request timeout:** 10s (standard HTTP requests)

---

### 8. Authentication

All endpoints require authentication via Clerk JWT token:

**Header:**

```
Authorization: Bearer <clerk_jwt_token>
```

**WebSocket:**

- JWT token in `Authorization` header on connection
- Or `token` query parameter: `wss://api.charismaai.app/ws?token=<jwt_token>`

---

### 9. Rate Limiting

**Per User Limits:**

- **Upload:** 10 requests/hour
- **Generate Suggestions:** 20 requests/hour
- **Message CRUD:** 100 requests/hour
- **General API:** 1000 requests/hour

**Rate Limit Headers:**

```
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 7
X-RateLimit-Reset: 1234567890  // Unix timestamp (seconds)
```

**Error Response (429 Too Many Requests):**

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later.",
    "retryAfter": 3600 // seconds until limit resets
  }
}
```

---

### 10. Retry Strategies

#### 10.1 Client-side retries

**Upload Failure:**

- **Attempts:** 3
- **Delays:** 0s (immediate), 2s, 5s
- **Conditions:** Network errors, 5xx responses

**API 5xx Errors:**

- **Attempts:** 2
- **Delays:** 1s, 3s
- **Conditions:** Server errors (500, 502, 503, 504)

**WebSocket Disconnect:**

- **Attempts:** Infinite
- **Strategy:** Exponential backoff (1s, 2s, 4s, 8s, 16s, 30s max)
- **Conditions:** Connection lost, network issues

#### 10.2 Server-side retries (BullMQ)

**OCR Failure:**

- **Attempts:** 2
- **Delay:** 5s between attempts
- **Conditions:** Gemini API errors, transient failures

**AI Generation:**

- **Attempts:** 3
- **Delays:** 10s, 30s, 60s
- **Conditions:** OpenAI/Anthropic API errors, rate limits

---

### 11. CORS Configuration

**Allowed Origins:**

- `https://app.charismaai.com` (production)
- `https://www.charismaai.app` (production)
- `http://localhost:8081` (development)
- `http://localhost:3000` (development)

**Allowed Methods:**

- `GET`, `POST`, `PATCH`, `DELETE`, `OPTIONS`

**Allowed Headers:**

- `Authorization`
- `Content-Type`
- `X-Requested-With`

**Credentials:**

- `Access-Control-Allow-Credentials: true`

---

### 12. Notes

- All timestamps are Unix timestamps in **milliseconds** (frontend format)
- Backend stores as `DateTime`, converts in service layer
- `ocrResult` is temporary - removed after user confirms (set to `null`)
- Soft delete: `isActive = false` + `deletedAt` set
- WebSocket auto-reconnects on disconnect (infinite attempts, exponential backoff)

---

### 13. References

- Frontend Schema: `client/src/schemas/session.schema.ts`
- Prisma Schema: `docs/backend/sessions/architecture/PRISMA_SCHEMA_DRAFT.md`
- Upload Flow: `docs/backend/sessions/architecture/UPLOAD_FLOW.md`
