## Knowledge Base Maintenance – Sessions Module

### 1. Purpose and Scope

This document standardizes how we maintain and evolve the **Sessions knowledge base** (RAG chunks) after initial ingestion.
It is intended for backend and content engineers who:

- Edit architect JSON chunks.
- Run ingestion scripts.
- Need to reason about when embeddings must be regenerated and how to avoid unnecessary cost.

The focus is on:

- When and how to re-ingest chunks.
- Versioning and soft delete conventions.
- CLI options and future enhancements for ingestion.
- How to verify that the database state matches the JSON source of truth.

---

### 2. Current Behavior (Upsert Logic)

The ingestion pipeline already supports **idempotent upserts** on `chunk_id`:

```typescript
// ingestion.service.ts - ingestChunk()
ON CONFLICT (chunk_id) DO UPDATE SET
  version = EXCLUDED.version,
  source = EXCLUDED.source,
  is_active = EXCLUDED.is_active,
  text_content = EXCLUDED.text_content,
  embedding_text = EXCLUDED.embedding_text,
  embedding = EXCLUDED.embedding,
  updated_at = NOW()
```

Implications:

- Re-running ingestion for an existing `chunk_id` updates the row.
- New chunks are inserted.
- By default, the embedding is regenerated on every run, even if `embeddingText` is unchanged (this is what we optimize below).

---

### 3. When to Re-Ingest Chunks

Re-ingestion is required when any fields that affect retrieval or model behavior change.

Re-ingest when:

- You change `embeddingStrings.semantic` (`embeddingText`) – **embedding must be regenerated**.
- You change `languageVariants.en.text` (`textContent`) – **display text must be updated**.
- You change metadata (phase, goal, type, technique, style, context) – **metadata columns must be updated**.
- You change core guidance fields (for example, `coreStrategy`, `guidance`) that are mapped to DB fields.
- You add a new chunk – **insert new row**.
- You deactivate a chunk (`isActive: false`) – **update `is_active`**.

Re-ingestion is _not_ required when:

- Only `responseOptions` or `pitfalls` change and they are not mapped into `KnowledgeChunk`.
- Only comments/formatting change in the JSON file without impacting parsed fields.

---

### 4. Embedding Regeneration Optimization

#### 4.1 Problem

By default, ingestion regenerates embeddings for all chunks on every run:

- This is **expensive** in OpenAI usage.
- It slows down ingestion for large knowledge bases.

#### 4.2 Approach

Only regenerate the embedding when `embeddingText` has actually changed:

- Compare the new `embeddingText` from JSON with the existing `embedding_text` column.
- If they are equal, reuse the existing `embedding` from the database.
- If not, call the embedding service and update both `embedding_text` and `embedding`.

Pseudo-implementation (core idea):

```typescript
// ingestion.service.ts - ingestChunk()
const ingestChunk = async (
  architectChunk: ArchitectChunk,
  options?: { skipEmbeddingIfUnchanged?: boolean }
): Promise<KnowledgeChunk> => {
  const parsed = parseChunk(architectChunk);

  let shouldRegenerateEmbedding = true;
  let existingEmbedding: number[] | null = null;

  if (options?.skipEmbeddingIfUnchanged) {
    const existing = await prisma.$queryRaw<
      Array<{ embedding_text: string; embedding: string }>
    >`
      SELECT embedding_text, embedding::text as embedding
      FROM knowledge_chunks
      WHERE chunk_id = ${parsed.chunkId}
    `;

    if (
      existing.length > 0 &&
      existing[0].embedding_text === parsed.embeddingText
    ) {
      shouldRegenerateEmbedding = false;
      const vectorStr = existing[0].embedding.replace(/[\[\]]/g, "");
      existingEmbedding = vectorStr.split(",").map(Number);
    }
  }

  const embedding = shouldRegenerateEmbedding
    ? await embeddingService.embed(parsed.embeddingText)
    : existingEmbedding!;

  // Upsert using parsed + embedding...
};
```

Use this optimization via a CLI flag (see section 7).

---

### 5. Versioning and Soft Delete Conventions

#### 5.1 Version tracking

Best practices:

- Use `metadata.version` in JSON as the canonical chunk version.
- Bump the version **whenever you change a chunk** in a way that impacts behavior.
- Use semantic versioning (`major.minor.patch`) as a guideline, e.g. `"2.1.0"`.

Example:

```json
{
  "metadata": {
    "chunk_id": "arch_attraction_001",
    "version": "2.1.0",
    "source": "MANUAL_DRAFT"
  }
}
```

#### 5.2 Soft delete (`isActive`)

Rules:

- Do **not** hard-delete rows from `knowledge_chunks`.
- Use `isActive: false` in JSON to deactivate a chunk.
- RAG queries must always filter on `is_active = true`.

Benefits:

- Enables rollback and historical audit.
- Keeps the knowledge base consistent with previous runs.

---

### 6. Selective Re-Ingestion

You do not need to re-ingest the entire knowledge base for every change.

Examples:

```bash
# Re-ingest only Phase 1 chunks
npm run ingest:knowledge-base rag/architect/1_attraction

# Re-ingest a single file
npm run ingest:knowledge-base rag/architect/1_attraction/arch_attraction_001.json

# Re-ingest all chunks
npm run ingest:knowledge-base rag/architect
```

Combine this with `--skip-unchanged` for faster runs.

---

### 7. Developer Workflow – Updating Chunks

#### 7.1 Scenario 1 – Updating an existing chunk

```bash
# 1. Edit the JSON file
vim rag/architect/1_attraction/arch_attraction_001.json

# 2. Bump the version in metadata
{
  "metadata": {
    "chunk_id": "arch_attraction_001",
    "version": "2.1.0",
    ...
  }
}

# 3. Re-ingest only that file (or the directory)
npm run ingest:knowledge-base rag/architect/1_attraction/arch_attraction_001.json

# Or re-ingest the directory (if many chunks changed)
npm run ingest:knowledge-base rag/architect/1_attraction
```

#### 7.2 Scenario 2 – Adding a new chunk

```bash
# 1. Create a new JSON file
cp rag/architect/1_attraction/arch_attraction_001.json \
   rag/architect/1_attraction/arch_attraction_077.json

# 2. Update metadata (chunk_id, version, etc.)
vim rag/architect/1_attraction/arch_attraction_077.json

# 3. Ingest the new chunk
npm run ingest:knowledge-base rag/architect/1_attraction/arch_attraction_077.json
```

#### 7.3 Scenario 3 – Deactivating a chunk (soft delete)

```bash
# 1. Edit the JSON file
vim rag/architect/1_attraction/arch_attraction_001.json

# 2. Set isActive to false
{
  "metadata": {
    "chunk_id": "arch_attraction_001",
    "isActive": false,
    ...
  }
}

# 3. Re-ingest
npm run ingest:knowledge-base rag/architect/1_attraction/arch_attraction_001.json
```

#### 7.4 Scenario 4 – Bulk update (multiple chunks)

```bash
# 1. Edit multiple JSON files (e.g. all Phase 1 chunks)

# 2. Re-ingest the directory
npm run ingest:knowledge-base rag/architect/1_attraction

# Or re-ingest the entire knowledge base
npm run ingest:knowledge-base rag/architect
```

---

### 8. Ingestion CLI Enhancements (Current and Planned)

#### 8.1 `--skip-unchanged` (optimized embeddings)

Add a flag to skip embedding regeneration when `embeddingText` is unchanged:

```typescript
// scripts/ingest-knowledge-base.ts

async function main() {
  const knowledgeBasePath = process.argv[2] || defaultPath;
  const skipUnchanged = process.argv.includes("--skip-unchanged");

  // ...

  for (const filePath of jsonFiles) {
    await ingestionService.ingestChunk(architectChunk, {
      skipEmbeddingIfUnchanged: skipUnchanged,
    });
  }
}
```

Usage:

```bash
npm run ingest:knowledge-base -- --skip-unchanged rag/architect
```

#### 8.2 `--dry-run` (preview changes)

Supports a preview-only mode:

```typescript
const dryRun = process.argv.includes("--dry-run");

if (dryRun) {
  logger.info("DRY RUN MODE - No changes will be made");
  // compute and display what would change
}
```

Usage:

```bash
npm run ingest:knowledge-base -- --dry-run rag/architect
```

#### 8.3 `--diff` (show what changed) – planned

Goal:

- Compare existing DB chunks with new JSON.
- Display differences for version, text, embedding text, and metadata.

#### 8.4 Selective ingestion filters – planned

CLI filters for phase/goal:

```typescript
const onlyPhase = process.argv
  .find((arg) => arg.startsWith("--phase="))
  ?.split("=")[1];
const onlyGoal = process.argv
  .find((arg) => arg.startsWith("--goal="))
  ?.split("=")[1];

const filteredChunks = jsonFiles.filter((file) => {
  const chunk = JSON.parse(readFileSync(file, "utf-8"));
  // filter by parsed tags.phase / tags.goal
  // ...
  return true;
});
```

Usage examples:

```bash
npm run ingest:knowledge-base -- --phase=1 rag/architect
npm run ingest:knowledge-base -- --goal=attraction rag/architect
```

#### 8.5 Progress bar for batch ingestion

Use a CLI progress bar to show ingestion progress:

```typescript
import cliProgress from "cli-progress";

const progressBar = new cliProgress.SingleBar(
  {
    format:
      "Ingestion Progress |{bar}| {percentage}% | {value}/{total} chunks | ETA: {eta}s",
  },
  cliProgress.Presets.shades_classic
);

progressBar.start(jsonFiles.length, 0);

for (const filePath of jsonFiles) {
  await ingestionService.ingestChunk(architectChunk);
  progressBar.increment();
}

progressBar.stop();
```

---

### 9. Monitoring and Verification

#### 9.1 Verify ingested chunks

Via Prisma Studio:

```bash
npm run db:studio
# Inspect knowledge_chunks table
# Filter by updated_at to see recent changes
```

Via SQL:

```sql
SELECT
  chunk_id,
  version,
  is_active,
  updated_at,
  CASE
    WHEN embedding IS NOT NULL THEN 'Has embedding'
    ELSE 'Missing embedding'
  END AS embedding_status
FROM knowledge_chunks
ORDER BY updated_at DESC
LIMIT 20;
```

#### 9.2 Check that embeddings are up to date

```sql
SELECT
  chunk_id,
  version,
  updated_at,
  embedding IS NOT NULL AS has_embedding
FROM knowledge_chunks
WHERE updated_at > NOW() - INTERVAL '1 day'
ORDER BY updated_at DESC;
```

#### 9.3 Check active chunks per phase

```sql
SELECT
  phase,
  COUNT(*) AS active_chunks
FROM knowledge_chunks
WHERE is_active = TRUE
GROUP BY phase
ORDER BY phase;
```

---

### 10. Operational Notes

#### 10.1 Embedding regeneration cost

- Embeddings are regenerated **whenever `embeddingText` changes**.
- This is the main cost driver in ingestion.
- Use `--skip-unchanged` whenever possible to avoid unnecessary calls.

#### 10.2 Versioning

- Always bump `metadata.version` when you change a chunk.
- This supports auditing, reproducibility, and communication across the team.

#### 10.3 Soft delete

- Use `isActive: false` instead of deleting records.
- Ensures you can restore or inspect old content if needed.

#### 10.4 Backups

- For large/bulk ingestion runs, take a backup first:

```bash
pg_dump $DATABASE_URL -t knowledge_chunks > knowledge_chunks_backup.sql
```

---

### 11. Summary

Current flow:

1. Edit the architect JSON file(s).
2. Run `npm run ingest:knowledge-base <path>`.
3. Ingestion upserts into `knowledge_chunks` (with optional embedding optimization).

Key improvements and tooling:

- Optimized ingestion path (`--skip-unchanged`).
- Dry-run mode for safe previews.
- Progress bar for UX.
- Planned diff and selective ingestion filters.

Best practices:

- Always bump `metadata.version` when changing a chunk.
- Prefer soft delete (`isActive: false`) over hard delete.
- Re-ingest only what changed, and verify the DB state afterward.
