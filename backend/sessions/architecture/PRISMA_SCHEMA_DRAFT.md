## Sessions Prisma Schema – Design Reference

### 1. Purpose and Scope

This document captures the Prisma schema design for the **Sessions** feature:

- `Session` – main session entity.
- `Message` – chat messages within a session.
- `KnowledgeChunk` – RAG knowledge base chunks (used by the RAG pipeline).

It was originally written as a Week 0 draft and now serves as a reference for:

- Backend schema evolution.
- Frontend–backend enum and field alignment.
- Indexing, relationships, and migration notes.

Frontend schema reference: `client/src/schemas/session.schema.ts`.

---

### 2. Complete Schema

```prisma
// ============================================
// SESSION MODEL
// ============================================

enum SessionType {
  SCREENSHOT
  TEXT
}

enum SessionStatus {
  PENDING_USER_INPUT
  PREVIEWING_SCREENSHOT
  UPLOADING
  ANALYSIS_IN_PROGRESS
  VERIFYING_OCR
  COMPLETE
  ERROR
}

model Session {
  id            String        @id @default(cuid())
  userId        String        @map("user_id")

  // Core fields (must match frontend)
  type          SessionType
  status        SessionStatus @default(PENDING_USER_INPUT)
  name          String?       // Optional session name

  // Screenshot-specific
  screenshotUrl String?       @map("screenshot_url") // R2 URL after upload
  ocrResult     Json?        @map("ocr_result") // Temporary, before verification (array format)

  // Timestamps (frontend expects Unix timestamps, we store DateTime)
  createdAt     DateTime      @default(now()) @map("created_at")
  updatedAt     DateTime      @updatedAt @map("updated_at")
  lastActivity  DateTime      @default(now()) @map("last_activity") // For sorting
  deletedAt     DateTime?     @map("deleted_at") // Soft delete

  // State flags (frontend expects these)
  isActive      Boolean       @default(true) @map("is_active")
  isPinned      Boolean       @default(false) @map("is_pinned")

  // Relationships
  user          User          @relation(fields: [userId], references: [id], onDelete: Cascade)
  messages      Message[]

  // Indexes (performance)
  @@index([userId, createdAt])
  @@index([userId, lastActivity]) // For sorting by last activity
  @@index([userId, status]) // IMPORTANT: for filtering by status
  @@index([deletedAt]) // For soft delete queries
  @@map("sessions")
}

// ============================================
// MESSAGE MODEL
// ============================================

enum MessageAuthor {
  USER
  CONTACT
  AI
}

enum MessageSentiment {
  POSITIVE
  NEGATIVE
  NEUTRAL
}

model Message {
  id            String          @id @default(cuid())
  sessionId     String          @map("session_id")

  // Core fields (must match frontend)
  text          String          @db.Text // For long AI suggestions
  author        MessageAuthor
  timestamp     DateTime        @default(now()) // Unix timestamp in milliseconds (stored as DateTime)

  // Optional fields
  sentiment     MessageSentiment?
  collectionIds String[]        @default([]) @map("collection_ids") // For bookmarks (collections)
  // Note: isLiked, isDisliked, isPinned, usedCount are calculated from UserMessageInteraction
  // This keeps it consistent with Magic Openers pattern

  // AI suggestion fields
  title         String?         // Custom title
  tag           String?         // Tag (e.g., "Attraction", "Confidence")
  analysis      String?         @db.Text // "Why this works?" explanation (can be long)

  // Relationships
  session       Session         @relation(fields: [sessionId], references: [id], onDelete: Cascade)
  interactions  UserMessageInteraction[]

  // Indexes
  @@index([sessionId, timestamp]) // For ordering messages within session
  @@index([sessionId, author]) // For filtering by author
  @@map("messages")
}

// ============================================
// USER MESSAGE INTERACTION MODEL
// ============================================

model UserMessageInteraction {
  id        String            @id @default(cuid())
  userId    String            @map("user_id")
  messageId String            @map("message_id")
  type      InteractionType   // LIKED, DISLIKED, PINNED, COPIED_TO_CLIPBOARD, etc.
  createdAt DateTime          @default(now()) @map("created_at")
  message   Message           @relation(fields: [messageId], references: [id], onDelete: Cascade)
  user      User              @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId, messageId, type]) // One interaction type per user per message
  @@index([userId, createdAt])
  @@index([messageId, type]) // For calculating usedCount (COPIED_TO_CLIPBOARD)
  @@map("user_message_interactions")
}

// ============================================
// KNOWLEDGECHUNK MODEL (RAG)
// ============================================

model KnowledgeChunk {
  id            String   @id @default(cuid())
  chunkId       String   @unique @map("chunk_id") // from metadata.chunk_id (e.g., "arch_attraction_001")
  version       String   // from metadata.version (e.g., "2.0.0")
  source        String   // from metadata.source (e.g., "MANUAL_DRAFT")
  isActive      Boolean  @default(true) @map("is_active")

  // Content
  textContent   String   @db.Text @map("text_content") // from languageVariants.en.text
  embeddingText String   @db.Text @map("embedding_text") // from embeddingStrings (combined)
  embedding     Unsupported("vector(1536)")? // OpenAI embedding (pgvector)

  // Metadata
  phase         Int?     // from tags.phase[0] → parse number (1, 2, 3, 4)
  goal          String?  // from tags.goal[0] (attraction, comfort, commitment, closing)
  type          String?  // from tags.type[0] (opener, closing, response)
  techniques    String[] // from tags.technique
  context       String[] // from tags.context

  // Additional
  situation     String?  // from situation
  primaryIntent String?  @map("primary_intent") // from guidance.primary_intent
  riskLevel     String?  @map("risk_level") // from guidance.risk_level

  createdAt     DateTime @default(now()) @map("created_at")
  updatedAt     DateTime @updatedAt @map("updated_at")

  // Indexes
  @@index([phase, goal]) // For RAG filtering by phase/goal
  @@index([isActive]) // For filtering active chunks
  @@index([chunkId]) // For lookup by chunkId
  @@map("knowledge_chunks")
}

// ============================================
// USER MODEL UPDATE
// ============================================

// Add to existing User model:
// sessions Session[]
// messageInteractions UserMessageInteraction[]
```

---

### 3. Key Design Decisions

#### 3.1 Enums vs. strings

**Decision:** Use Prisma enums for type safety

- `SessionType`: `SCREENSHOT` | `TEXT`
- `SessionStatus`: Matches frontend enum exactly
- `MessageAuthor`: `USER` | `CONTACT` | `AI`
- `MessageSentiment`: `POSITIVE` | `NEGATIVE` | `NEUTRAL`

**Frontend Mapping:**

- Frontend uses lowercase: `'screenshot'` → Prisma: `SCREENSHOT`
- Frontend uses snake_case: `'pending_user_input'` → Prisma: `PENDING_USER_INPUT`

#### 3.2 Timestamps

**Decision:** Store as `DateTime`, convert to Unix timestamp in API layer

**Frontend expects:**

- `createdAt: number` (Unix timestamp in milliseconds)
- `lastActivity: number` (Unix timestamp in milliseconds)
- `timestamp: number` (Unix timestamp in milliseconds for messages)

**Backend stores:**

- `DateTime` (PostgreSQL timestamp)
- Convert in service layer: `date.getTime()` for frontend

#### 3.3 `ocrResult` format

**Decision:** Store as `Json?` (PostgreSQL JSONB)

**Frontend expects:**

```typescript
ocrResult?: Array<{
  id: string;
  text: string;
  author: "user" | "contact" | "ai";
}>
```

**Backend stores:**

- `Json?` (can be `null` or array)
- Validate with Zod before storing
- Remove after user confirms (set to `null`)

#### 3.4 `lastActivity` field

**Decision:** Stored field, updated on:

- Session creation
- Message addition
- Status change to `COMPLETE`
- Any user interaction

**Alternative considered:** Computed field (max of createdAt, updatedAt, last message timestamp)

- **Rejected:** Need explicit control for sorting

#### 3.5 Soft delete strategy

**Decision:** Use `deletedAt` + `isActive` flags

**Reasoning:**

- `deletedAt`: For hard delete queries (permanent removal)
- `isActive`: For frontend filtering (matches frontend schema)
- Both needed for different use cases

#### 3.6 `KnowledgeChunk` embeddings

**Decision:** Use `pgvector` extension for vector similarity search

**Setup required:**

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

**Vector dimension:** 1536 (OpenAI text-embedding-3-small)

---

### 4. Index Strategy

#### 4.1 Session indexes

1. `@@index([userId, createdAt])`

   - **Purpose:** List sessions by user, newest first
   - **Query:** `WHERE userId = ? ORDER BY createdAt DESC`

2. `@@index([userId, lastActivity])`

   - **Purpose:** List sessions by user, most active first
   - **Query:** `WHERE userId = ? ORDER BY lastActivity DESC`

3. `@@index([userId, status])` ⭐ **IMPORTANT**

   - **Purpose:** Filter sessions by status
   - **Query:** `WHERE userId = ? AND status = ?`
   - **Use case:** Get all active sessions, get all complete sessions

4. `@@index([deletedAt])`
   - **Purpose:** Soft delete queries
   - **Query:** `WHERE deletedAt IS NULL`

#### 4.2 Message indexes

1. `@@index([sessionId, timestamp])`

   - **Purpose:** Order messages within session
   - **Query:** `WHERE sessionId = ? ORDER BY timestamp ASC`

2. `@@index([sessionId, author])`
   - **Purpose:** Filter messages by author
   - **Query:** `WHERE sessionId = ? AND author = ?`

#### 4.3 KnowledgeChunk indexes

1. `@@index([phase, goal])`

   - **Purpose:** RAG filtering by phase/goal
   - **Query:** `WHERE phase = ? AND goal = ? AND isActive = true`

2. `@@index([isActive])`

   - **Purpose:** Filter active chunks only
   - **Query:** `WHERE isActive = true`

3. `@@index([chunkId])`
   - **Purpose:** Lookup by chunkId
   - **Query:** `WHERE chunkId = ?`

---

### 5. Relationships

```
User (1) ──→ (N) Session
Session (1) ──→ (N) Message
```

**Cascade Delete:**

- Delete User → Delete all Sessions → Delete all Messages
- Delete Session → Delete all Messages

**No cascade:**

- KnowledgeChunk is independent (no user relationship)

---

### 6. Frontend Schema Verification

#### 6.1 `SessionStatus` enum match

**Frontend:** `client/src/schemas/session.schema.ts:8-16`

```typescript
"pending_user_input";
"previewing_screenshot";
"uploading";
"analysis_in_progress";
"verifying_ocr";
"complete";
"error";
```

**Backend Prisma:**

```prisma
PENDING_USER_INPUT
PREVIEWING_SCREENSHOT
UPLOADING
ANALYSIS_IN_PROGRESS
VERIFYING_OCR
COMPLETE
ERROR
```

**Mapping:** Convert in service layer (snake_case → UPPER_SNAKE_CASE)

#### 6.2 `MessageAuthor` enum match

**Frontend:** `'user' | 'contact' | 'ai'`  
**Backend:** `USER | CONTACT | AI`

**Mapping:** Convert in service layer (lowercase → UPPERCASE)

---

### 7. Migration Notes

#### 7.1 Add pgvector extension

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

#### 7.2 Create migration

```bash
npx prisma migrate dev --name add_sessions_feature
```

#### 7.3 Create vector index (manual) ⚠️

**Important:** Prisma cannot auto-create IVFFlat index. Create manually after migration:

```sql
-- Create vector index for similarity search
CREATE INDEX knowledge_chunks_embedding_idx
ON knowledge_chunks
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 10);

-- Note: Adjust 'lists' parameter based on data size:
-- - Small dataset (< 10K chunks): lists = 10
-- - Medium dataset (10K-100K): lists = 100
-- - Large dataset (> 100K): lists = 1000
```

**Why IVFFlat?**

- Fast approximate similarity search
- `vector_cosine_ops` for cosine similarity (standard for embeddings)
- `lists` parameter: higher = more accurate but slower

**When to create:** After Week 5 ingestion (when embeddings are populated)

#### 7.4 Verify indexes

```sql
-- Check indexes were created
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename IN ('sessions', 'messages', 'knowledge_chunks');
```

---

### 8. Historical Next Steps (Week 1 Plan)

1. ✅ Review this draft with team
2. ✅ Verify enum mappings match frontend
3. ✅ Create Prisma migration
4. ✅ **Manually create vector index** (IVFFlat) for KnowledgeChunk embeddings
5. ✅ Update User model to include `sessions Session[]` and `messageInteractions UserMessageInteraction[]`
6. ✅ Test schema with sample data
7. ✅ Verify indexes are created correctly

---

### 9. Important Notes

- **Do NOT commit this to schema.prisma yet** - this is Week 0 draft
- **Verify enum values** match frontend exactly before migration
- **Test timestamp conversion** (DateTime → Unix timestamp) in service layer
- **ocrResult** is temporary - should be `null` after user confirms
- **KnowledgeChunk** will be added in Week 5 (RAG ingestion)

---

### 10. References

- Frontend Schema: `client/src/schemas/session.schema.ts`
- Chunk Structure: `docs/backend/sessions/CHUNK_STRUCTURE.md`
- API Contract: `docs/backend/sessions/SESSIONS_API_CONTRACT.md`
