## Sessions Upload Strategy – Cost Analysis

### 1. Purpose and Scope

This document analyzes the **infrastructure cost** of the Sessions upload strategy for the MVP and early growth phases.
It focuses on:

- Cloudflare R2 storage and operations.
- Background jobs and Redis (BullMQ).
- WebSocket real-time updates.
- OCR (Gemini Vision).
- Embeddings (OpenAI `text-embedding-3-small`) used by Sessions/RAG.

Global AI cost tracking and rollups are documented in `docs/backend/analytics/COST_TRACKING_&_ANALYTICS.md`.

---

### 2. Upload Strategy Components

The upload strategy consists of:

1. Pre-signed URLs for direct R2 uploads from the client.
2. Background jobs (BullMQ) for asynchronous processing (download, OCR, status updates).
3. WebSocket for real-time session status updates.
4. OCR processing (Gemini Vision).
5. Embeddings for RAG ingestion and queries.

---

### 3. Cost Breakdown – MVP

#### 3.1 Cloudflare R2 storage

**R2 pricing (2025):**

- Storage: **$0.015 per GB/month**
- Class A operations (write): **$4.50 per million**
- Class B operations (read): **$0.36 per million**
- Egress: **FREE** (unlimited)

**MVP usage estimation:**

Storage:

- 100 sessions/day × 30 days ≈ **3,000 sessions/month**
- Average screenshot size ≈ **500 KB** per session
- Total storage: 3,000 × 0.5 MB ≈ **1.5 GB/month**

Operations:

- Class A (write): 3,000 uploads/month ≈ **0.003M ops**
- Class B (read): 3,000 reads/month (for OCR) ≈ **0.003M ops**

Cost calculation:

- Storage: 1.5 GB × $0.015 ≈ **$0.023/month**
- Class A: 0.003M × $4.50 ≈ **$0.014/month**
- Class B: 0.003M × $0.36 ≈ **$0.001/month**
- Egress: **$0**

**Total R2 cost (MVP): ≈ $0.04/month**

---

#### 3.2 Background jobs (BullMQ + Redis)

Status: already analyzed in the general Redis/Upstash cost analysis.

- Upstash Redis: **$0/month** (free tier)
- BullMQ: **open source** (no direct cost)
- Commands: ~135K/month (within free tier)

**Total BullMQ/Redis cost (MVP): ≈ $0/month**

---

#### 3.3 WebSocket real-time updates

Status: covered by existing backend hosting (e.g. Railway).

- Backend hosting already supports WebSocket traffic.
- No additional per-connection fees at current scale.
- Bandwidth included in hosting plan.

**Total WebSocket cost (MVP): ≈ $0/month** (already covered)

---

#### 3.4 OCR processing (Gemini 2.5 Flash Vision)

Status: part of the existing AI stack.

Assumptions:

- Gemini 2.5 Flash Vision: **~$0.075 per 1M tokens**
- Average OCR per screenshot: **~500 tokens**
- 3,000 sessions/month × 500 tokens ≈ **1.5M tokens/month**

Cost:

- 1.5M tokens × $0.075/1M ≈ **$0.11/month**

---

#### 3.5 Embeddings (OpenAI `text-embedding-3-small`)

Status: used for RAG ingestion and queries.

- Model: `text-embedding-3-small` – **$0.02 per 1M tokens**

One-time ingestion (example):

- 76 chunks × ~500 tokens/chunk ≈ **38K tokens**
- 38K × $0.02/1M ≈ **$0.0008** (negligible)

Ongoing RAG queries:

- 3,000 sessions/month × 1 query × ~100 tokens ≈ **300K tokens/month**
- 300K × $0.02/1M ≈ **$0.006/month**

**Total embeddings cost (MVP): ≈ $0.01/month**

---

### 4. Total MVP Cost Summary

| Component                   | Monthly Cost | Notes                             |
| --------------------------- | ------------ | --------------------------------- |
| Cloudflare R2 storage       | ≈ $0.04      | 1.5 GB storage + Class A/B ops    |
| Upstash Redis (BullMQ)      | ≈ $0.00      | Free tier (~135K commands)        |
| WebSocket (backend hosting) | ≈ $0.00      | Included in existing backend host |
| OCR (Gemini 2.5 Flash)      | ≈ $0.11      | ~1.5M tokens/month                |
| Embeddings (OpenAI)         | ≈ $0.01      | RAG ingestion + queries           |
| Backend hosting (Railway)   | ≈ $5–20      | Existing backend hosting plan     |
| **TOTAL (MVP)**             | **~$5–20**   | Dominated by hosting, not upload  |

Upload-strategy-specific cost (R2 + OCR + embeddings + Redis/WebSocket) is roughly **$0.16/month** at MVP volumes.

---

### 5. Key Insights and Optimization Tips

#### 5.1 Cost-effective components

- **Cloudflare R2**

  - Free egress vs. AWS S3.
  - Very low storage cost at current scale.
  - Pay-per-use operations with negligible totals.

- **Upstash Redis**

  - Free tier sufficient for MVP.
  - Auto-scaling and serverless.

- **WebSocket**

  - No incremental cost beyond backend hosting.

- **AI (OCR + embeddings)**
  - Overall AI infra cost is low relative to base hosting at MVP scale.

#### 5.2 R2 storage optimizations

- Apply lifecycle policies (e.g. delete screenshots older than 30–60 days).
- Compress screenshots before upload if UX allows.
- Use R2 CDN integration for repeated reads if necessary.

#### 5.3 OCR cost optimizations

- Cache OCR results to avoid re-running on the same image.
- Limit retries for clearly invalid inputs.
- Prefer Gemini 2.5 Flash for cost-efficient OCR.

#### 5.4 Embedding cost optimizations

- Cache embeddings for repeated texts.
- Batch embedding requests for throughput and cost efficiency.
- Use `text-embedding-3-small` for cost-sensitive RAG tasks.

---

### 6. Scaling Projections

Assuming roughly linear scaling in usage:

#### 6.1 Phase 1 – MVP (0–1K users/day)

- R2: ≈ $0.04/month
- Redis: ≈ $0/month
- OCR: ≈ $0.11/month
- Embeddings: ≈ $0.01/month

**Total upload strategy cost:** ≈ **$0.16/month**

#### 6.2 Phase 2 – Growth (1K–10K users/day)

- R2: ≈ $0.50/month (≈15 GB)
- Redis: ≈ $5/month (Upstash pay-as-you-go)
- OCR: ≈ $1/month (≈15M tokens)
- Embeddings: ≈ $0.10/month

**Total upload strategy cost:** ≈ **$6.60/month**

#### 6.3 Phase 3 – Scale (10K+ users/day)

- R2: ≈ $5/month (≈150 GB)
- Redis: ≈ $20/month (managed Redis)
- OCR: ≈ $10/month (≈150M tokens)
- Embeddings: ≈ $1/month

**Total upload strategy cost:** ≈ **$36/month**

---

### 7. Final Summary

- Upload strategy incremental cost at MVP scale is **very low (~$0.16/month)**.
- The dominant recurring cost is backend hosting (Railway), not R2 or AI.
- The current design is cost-efficient and scales smoothly through early growth phases.

If needed, combine:

- Global AI cost tracking, rollups, and alerts (`COST_TRACKING_&_ANALYTICS.md`).
- R2 lifecycle and caching strategies described above.

---

### 8. Implementation Checklist

- [x] Create Cloudflare R2 bucket.
- [x] Configure R2 API keys and environment variables.
- [x] Implement pre-signed URL generation.
- [x] Configure BullMQ queues and Upstash Redis (or equivalent).
- [x] Test the upload + OCR flow end-to-end.

---

### 9. External References

- Cloudflare R2 Pricing – `https://developers.cloudflare.com/r2/pricing/`
- Upstash Redis Pricing – `https://upstash.com/pricing`
- Gemini 2.5 Flash Pricing – `https://ai.google.dev/pricing`
- OpenAI Embeddings Pricing – `https://openai.com/pricing`
