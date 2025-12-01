## Sessions WebSocket Protocol – Design

### 1. Purpose and Status

This document specifies the WebSocket protocol used by the **Sessions** feature for real-time status updates during upload, OCR processing, and related flows.
It is intended for backend engineers working on WebSocket handling, Redis/BullMQ integration, or client engineers integrating with the protocol.

- **Status:** Implemented; this document captures the intended contract and behavior.
- **Scope:** Connection details, authentication, message formats, reconnection strategy, security, performance, and testing.

For the HTTP-side flows, see `UPLOAD_FLOW.md` and `SESSIONS_API_CONTRACT.md`.

---

---

### 2. Connection

#### 2.1 URL

**Production:**

```
wss://api.charismaai.app/ws
```

**Development:**

```
ws://localhost:8080/ws
```

#### 2.2 Authentication

**Option 1: Header**

```
Authorization: Bearer <clerk_jwt_token>
```

**Option 2: Query Parameter**

```
wss://api.charismaai.app/ws?token=<clerk_jwt_token>
```

---

---

### 3. Client → Server Messages

### Authenticate

```json
{
  "action": "authenticate",
  "payload": {
    "token": "clerk_jwt_token"
  }
}
```

**Response:**

```json
{
  "type": "authenticated",
  "message": "Authentication successful"
}
```

### Subscribe to Session Updates

```json
{
  "action": "subscribe",
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

### Unsubscribe from Session Updates

```json
{
  "action": "unsubscribe",
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

---

---

### 4. Server → Client Messages

### Session Status Update

```json
{
  "type": "session_status",
  "sessionId": "session_123",
  "status": "analysis_in_progress",
  "message": "Analyzing screenshot...",
  "progress": 0.5
}
```

**Status Values:**

- `uploading`
- `analysis_in_progress`
- `verifying_ocr`
- `complete`
- `error`

**Progress:** Optional, 0.0 - 1.0

### Upload Progress

```json
{
  "type": "session_upload_progress",
  "sessionId": "session_123",
  "progress": 0.75
}
```

**Progress:** 0.0 - 1.0

### OCR Complete

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

### Error

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

---

### 5. Message Flow

### Upload & Analysis Flow

```
1. Client subscribes to session
   ↓
2. Client uploads file to R2
   ↓
3. Server sends: session_upload_progress (0.0 → 1.0)
   ↓
4. Server sends: session_status (uploading → analysis_in_progress)
   ↓
5. OCR processing (BullMQ worker)
   ↓
6. Server sends: session_status (verifying_ocr) + ocrResult
   ↓
7. User confirms OCR
   ↓
8. Server sends: session_status (complete)
```

---

---

### 6. Reconnection Strategy

#### 6.1 Client-side

**Strategy:** Exponential backoff

- **Attempts:** Infinite
- **Delays:** 1s, 2s, 4s, 8s, 16s, 30s (max)
- **Conditions:** Connection lost, network issues

#### 6.2 Server-side

- **Heartbeat:** 30s interval
- **Timeout:** 60s (no heartbeat = disconnect)
- **Auto-cleanup:** Remove subscriptions on disconnect

---

---

### 7. Security

#### 7.1 Authentication

- JWT token required on connection
- Token validated on each message
- Unauthorized connections rejected

#### 7.2 Authorization

- Users can only subscribe to their own sessions
- Server validates session ownership

---

---

### 8. Performance

### Connection Limits

- **Per User:** 1 active connection
- **Per Session:** Multiple subscribers allowed
- **Total Connections:** Limited by server capacity

### Message Rate

- **Status Updates:** ~1-5 messages per session
- **Progress Updates:** ~10-20 messages per upload
- **Total:** ~20-30 messages per session lifecycle

---

### 9. Testing

Integration tests live in:

- `backend/src/modules/sessions/__tests__/integration/websocket.test.ts`

Recommended test scenarios:

- Authentication success and failure.
- Subscription and unsubscription behavior.
- Session status updates.
- Upload progress events.
- Error handling (`session_error`, malformed messages).
- Reconnection and resubscription flow.

---

### 10. References

- [Sessions API Contract](../SESSIONS_API_CONTRACT.md#websocket-protocol) – WebSocket section of the HTTP API spec.
- [Upload Flow](./UPLOAD_FLOW.md) – End-to-end upload and OCR architecture.
- [RAG Design](./RAG.md) – RAG architecture (downstream consumer of session state).

---

### 11. Implementation Status (Historical)

- Week 0: Protocol design.
- Week 2: WebSocket server implementation.
- Week 6: Integration testing and stabilization.
