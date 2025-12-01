## Retrieval-Augmented Generation (RAG) – Sessions Module Design

### 1. Purpose and Status

This document describes the design and implementation of the **RAG (Retrieval-Augmented Generation)** pipeline used by the Sessions module.
It is intended for backend and data engineers who work on AI suggestions, the knowledge base, or performance tuning.

- **Status:** Implemented, iteratively evolving.
- **Scope:** Knowledge ingestion, embeddings, vector search, and prompt construction for RAG-enhanced suggestions.

#### 1.1 Related Documents (How Everything Connects)

This file is the **canonical architecture document** for the RAG system. The following documents provide deeper technical detail by domain:

- **Chunk structure:** `CHUNK_STRUCTURE.md` – JSON schema and field-level definitions for knowledge chunks.
- **Database schema:** `PRISMA_SCHEMA_DRAFT.md` – `KnowledgeChunk` model, indexes, and database design.
- **Knowledge base maintenance / ingestion operations:** `../knowledge-base/KNOWLEDGE_BASE_MAINTENANCE.md`.
- **Upload and preprocessing flow:** `UPLOAD_FLOW.md` – how raw user uploads and OCR feed into the RAG knowledge base.
- **Cost tracking and analysis:**
  - `../../analytics/COST_TRACKING_&_ANALYTICS.md` – global AI/RAG cost tracking patterns.
  - `../cost/UPLOAD_STRATEGY_COST_ANALYSIS.md` and `../cost/UPSTASH_COST_ANALYSIS.md` – detailed cost models and trade-offs.

---

### 2. High-Level Architecture

#### 2.1 Components

1. **Knowledge Base**
   - Source of truth for domain-specific “playbook” content.
   - Stored as JSON chunk files in `backend/rag/architect/`.
2. **Embedding Service**
   - Uses OpenAI `text-embedding-3-small` (1536 dimensions).
   - Provides embeddings for both chunks (offline) and queries (online).
3. **Vector Store**
   - PostgreSQL with `pgvector` extension.
   - `KnowledgeChunk.embedding` column stores `vector(1536)`.
4. **RAG Service**
   - Performs vector similarity search and combines results with session state.
5. **Prompt Builder**
   - Constructs RAG-enhanced prompts for the suggestion service.

---

### 3. Data Flow (Ingestion and Query)

1. **Ingestion time**

   1. Read JSON chunks from `backend/rag/architect/`.
   2. Parse into internal chunk structure.
   3. Generate embeddings (OpenAI text-embedding-3-small).
   4. Persist chunks into `KnowledgeChunk` table (including `embedding`).
   5. Create / update vector index (for example, IVFFlat).

2. **Query time (per suggestion request)**
   1. Build a query string from the **entire session conversation** (see section on prompt building).
   2. Generate an embedding for the query.
   3. Perform vector similarity search over `knowledge_chunks` (with optional filters).
   4. Retrieve top N chunks.
   5. Build an AI prompt that includes:
      - System instructions.
      - Retrieved chunks as context.
      - Full conversation history.
      - User preferences (style, twist, language).
   6. Call the AI provider (OpenAI / Anthropic) to generate the suggestion.

---

### 4. Knowledge Base Structure

The knowledge base is defined by a chunk format documented in `CHUNK_STRUCTURE.md`.

Key fields:

- `metadata.chunk_id` – Unique identifier (for example, `arch_attraction_001`).
- `languageVariants.en.text` – User-facing text content.
- `embeddingStrings.semantic` – Canonical semantic text used to compute embeddings.
- `tags.phase`, `tags.goal`, `tags.type` – Metadata used to pre-filter chunks before vector search.

---

### 5. Vector Search

#### 5.1 Embedding model

- Model: OpenAI `text-embedding-3-small`.
- Dimensions: 1536.
- Approximate cost: ~$0.02 per 1M tokens.

#### 5.2 Similarity metric and index

- Metric: cosine similarity (`vector <-> vector`).
- Index: IVFFlat (approximate nearest neighbors).
- Recommended lists:
  - < 10K chunks: `lists = 10`.
  - 10K–100K chunks: `lists = 100`.
  - > 100K chunks: `lists = 1000`.

#### 5.3 Query process

1. Extract query text from **all messages** in the session (not only initial OCR).
2. Generate an embedding for this query text.
3. Execute vector search with optional filters:

   ```sql
   SELECT chunk_id,
          text_content,
          embedding <-> query_embedding AS distance
   FROM knowledge_chunks
   WHERE is_active = TRUE
     AND (phase = :phase OR :phase IS NULL)
     AND (goal  = :goal  OR :goal  IS NULL)
   ORDER BY distance
   LIMIT 5;
   ```

---

### 6. Filtering Strategy

Filters are used to keep RAG results relevant and to reduce noise.

#### 6.1 Phase filtering

- Phase is inferred from the conversation (via phase detection).
- Example mapping:
  - Phase 1: attraction.
  - Phase 2: comfort.
  - Phase 3: commitment.
  - Phase 4: closing.

`KnowledgeChunk.phase` can be used to restrict chunks to the current or target phase.

#### 6.2 Goal filtering

Example values:

- `attraction`
- `comfort`
- `commitment`
- `closing`

#### 6.3 Type filtering

Examples:

- `opener`
- `closing`
- `response`

`KnowledgeChunk.type` (or equivalent tag) can be used to limit which chunk types are retrieved for different suggestion flows.

---

### 7. Prompt Building

Implementation: `backend/src/modules/sessions/prompts/rag-prompt.ts`.

Prompt structure:

1. **System message**
   - Sets assistant persona, tone, and safety constraints.
2. **Relevant chunks (top N)**
   - RAG results, injected as contextual knowledge.
3. **Conversation history**
   - All user/AI messages in the session (not limited to initial OCR).
4. **User preferences**
   - `aiStyle`, `customTwist`, language.
5. **Task instructions**
   - Clear instructions for how many suggestions to generate and in what format.

The prompt builder always uses the full conversation history, which allows:

- Iterative suggestion flows.
- Each new suggestion to leverage previous RAG calls and user messages.

---

### 8. Ingestion Process

#### 8.1 Steps

1. Read JSON files from `backend/rag/architect/`.
2. Parse chunk structure (see `CHUNK_STRUCTURE.md`).
3. Validate chunks (required fields, tag consistency).
4. Generate embeddings in batches (OpenAI text-embedding-3-small).
5. Store in database (`KnowledgeChunk` model).
6. Create or refresh the vector index (manual SQL).

#### 8.2 Error handling

- Invalid chunks:
  - Skip and log for review (including file name and chunk ID).
- Embedding failures:
  - Retry on transient errors.
  - Log permanently failing chunks for manual inspection.
- Batch processing:
  - Use efficient batch sizes (for example, 50–200 chunks).

---

### 9. Performance

#### 9.1 Index tuning

For IVFFlat:

- Small dataset (< 10K): `lists = 10`.
- Medium dataset (10K–100K): `lists = 100`.
- Large dataset (> 100K): `lists = 1000`.

These values should be reviewed as the knowledge base grows.

#### 9.2 Query optimization

Recommended practices:

- Limit results (typically top 5–10 chunks).
- Filter by `phase` / `goal` before search when possible.
- Cache:
  - Query embeddings for repeated prompts.
  - RAG results for short periods if the conversation has not changed.

---

### 10. Cost Analysis

See `UPLOAD_STRATEGY_COST_ANALYSIS.md` for a detailed cost breakdown.

Embedding costs (approximate):

- ~$0.02 per 1M tokens.
- ~76 chunks ≈ ~$0.0001 per ingestion.
- Query embeddings: on the order of ~$0.00001 per query.

RAG usage should be integrated with the global AI cost tracking (`analytics/COST_TRACKING_&_ANALYTICS.md`).

---

### 11. References

- [Chunk Structure](./CHUNK_STRUCTURE.md) – JSON chunk format.
- [Prisma Schema Draft](./PRISMA_SCHEMA_DRAFT.md) – `KnowledgeChunk` model.
- Sessions architecture docs:
  - [Upload Flow](./UPLOAD_FLOW.md)
  - [WebSocket Flow](./WEBSOCKET.md)
- Knowledge base operations: `../knowledge-base/KNOWLEDGE_BASE_MAINTENANCE.md`.
- Cost and analytics:
  - `../../analytics/COST_TRACKING_&_ANALYTICS.md`
  - `../cost/UPLOAD_STRATEGY_COST_ANALYSIS.md`
  - `../cost/UPSTASH_COST_ANALYSIS.md`

---

### 12. Implementation Status (Historical)

- Planning and design ✅
- Ingestion and initial RAG implementation ✅
  - Vector database setup (pgvector + `KnowledgeChunk` model).
  - Embedding service (OpenAI text-embedding-3-small).
  - Ingestion service (recursive directory ingestion).
  - RAG service (vector similarity search).
  - Unit tests (embedding, RAG service).
  - Integration tests (RAG service).
  - Session service prepared for RAG integration.
  - Knowledge base ingestion execution (ready to run).
- AI suggestions with RAG context – rolled out.
- Ongoing testing, tuning, and optimization.
