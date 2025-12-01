## Upstash Redis – Cost Analysis for Sessions MVP

### 1. Purpose and Scope

This document evaluates **Upstash Redis** as the primary Redis provider for the Sessions feature MVP, with a comparison to a fixed Redis instance (for example, Railway Redis).
It focuses on:

- Pricing and free tier coverage.
- Estimated command usage for Sessions (jobs, WebSocket, rate limiting, caching).
- Cost comparison: Upstash (serverless) vs. fixed Redis.
- Recommended rollout path and migration triggers.

---

### 2. Summary Recommendation

**Use Upstash Redis for the MVP.**

Rationale:

- The free tier comfortably covers the expected MVP usage.
- Pay-as-you-go model avoids fixed monthly costs when traffic is low.
- Serverless model removes operational overhead (no servers to manage).
- Migration to a fixed Redis (for example, Railway) is straightforward if needed later.

---

### 3. Upstash Pricing (2025)

#### 3.1 Free tier (MVP-friendly)

- 500,000 commands/month – free.
- 256 MB storage – free.
- 10,000 concurrent connections – free.
- 10 GB data transfer/month – free.

#### 3.2 Pay-as-you-go (beyond free tier)

- ~$0.20 per 100,000 commands above the free tier.
- ~$0.10 per GB storage above 256 MB.
- Unlimited data transfer.

---

### 4. Estimated Usage – Sessions MVP

#### 4.1 Scenario 1 – BullMQ job queue

Commands per job (approximate):

- `LPUSH` (add job).
- `BRPOP` (get job).
- `SET` / `GET` (job status).
- `DEL` (cleanup).
- `EXPIRE` (TTL).

Estimation:

- 100 sessions/day × 3 jobs/session (upload, OCR, RAG) ≈ **300 jobs/day**.
- 300 jobs/day × 15 commands ≈ **4,500 commands/day**.
- Monthly: **~135,000 commands**.

#### 4.2 Scenario 2 – WebSocket pub/sub

Commands per message (approximate):

- `PUBLISH` (send update).
- `SUBSCRIBE`/`UNSUBSCRIBE` (connection lifecycle).

Estimation:

- 100 sessions/day × 5 updates/session ≈ **500 messages/day**.
- 500 messages/day × 3 commands ≈ **1,500 commands/day**.
- Monthly: **~45,000 commands**.

#### 4.3 Scenario 3 – Rate limiting

Commands per request (approximate):

- `INCR` (increment counter).
- `EXPIRE` (set TTL).

Estimation:

- 1,000 API requests/day × 1.5 commands ≈ **1,500 commands/day**.
- Monthly: **~45,000 commands**.

#### 4.4 Scenario 4 – Caching (optional)

Commands per cache operation:

- `GET` (read).
- `SET` (write with TTL).

Estimation:

- 500 cache operations/day × 2 commands ≈ **1,000 commands/day**.
- Monthly: **~30,000 commands**.

#### 4.5 Total MVP usage

- Daily: **~8,500 commands/day**.
- Monthly: **~255,000 commands/month**.
- Upstash free tier: **500,000 commands/month**.

Conclusion: MVP usage fits comfortably within the free tier; **expected Upstash cost for MVP is $0/month**.

---

### 5. Upstash vs. Fixed Redis (e.g. Railway)

#### 5.1 Upstash (serverless)

Pros:

- Free tier (500K commands/month, 256 MB) covers MVP.
- Pay-as-you-go: **$0.20 / 100K commands** above free tier.
- Auto-scaling and global replication.
- Serverless – no infrastructure management.

Cons:

- Possible cold-start latency (~50–100 ms) on the first request after idle.

Estimated cost for MVP:

- **≈ $0/month** (within free tier).

#### 5.2 Railway Redis (fixed instance)

Pros:

- Fixed, predictable cost (~$5–10/month).
- No cold starts; consistent latency.
- Full control over instance configuration.

Cons:

- Fixed cost even when idle.
- Manual scaling and plan management.

Estimated cost for MVP:

- **≈ $5–10/month** (fixed).

---

### 6. Recommendation and Migration Criteria

#### 6.1 Recommendation for MVP

Adopt **Upstash Redis** for production in the MVP phase:

- Cost: **$0/month** at projected usage.
- Setup effort: minimal (create account, provision database, set `REDIS_URL`).
- Operational overhead: near-zero (serverless).

#### 6.2 When to consider moving to Railway (or dedicated Redis)

Consider migrating to a fixed Redis instance if:

- Command volume consistently exceeds the Upstash free tier and pay-as-you-go costs become material.
- You require very low tail latency (<10 ms) and want to avoid cold-start impacts.
- You need a dedicated instance for compliance, networking, or data residency reasons.

---

### 7. Implementation Notes

#### 7.1 Environment configuration

```bash
# Production (Upstash)
REDIS_URL=redis://default:xxx@xxx.upstash.io:6379

# Development (local Docker, optional)
REDIS_URL=redis://localhost:6379
```

#### 7.2 Connection example (`ioredis`)

```typescript
import Redis from "ioredis";

const redis = new Redis(process.env.REDIS_URL, {
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  lazyConnect: true, // reduces impact of cold starts
});
```

#### 7.3 BullMQ configuration

```typescript
import { Queue } from "bullmq";
import { connection } from "./redis"; // shared ioredis connection

const uploadQueue = new Queue("session-upload", {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: "exponential",
      delay: 2000,
    },
  },
});
```

---

### 8. Scaling Path

#### 8.1 Phase 1 – MVP (0–1K users/day)

- Provider: Upstash (free tier).
- Cost: **$0/month**.
- Commands: **~255K/month**.

#### 8.2 Phase 2 – Growth (1K–10K users/day)

- Provider: Upstash pay-as-you-go.
- Cost: **~$5–15/month**.
- Commands: **~1–2M/month**.

#### 8.3 Phase 3 – Scale (10K+ users/day)

- Provider: Railway Redis or Upstash Pro.
- Cost: **~$20–50/month**.
- Commands: **5M+ commands/month**.

---

### 9. Final Decision and Action Items

**Decision:** Use **Upstash Redis** for the Sessions MVP.

- Cost: **$0/month** at expected usage.
- Setup time: ~5 minutes (create account, provision DB, set environment variable).
- Maintenance: minimal (serverless).
- Migration risk: low (easy to switch to Railway or another provider later).

Action items:

1. Create an Upstash account.
2. Provision a Redis database (free tier).
3. Add `REDIS_URL` to production environment variables.
4. Keep local Redis (Docker) for development if desired.
5. Test the connection and key flows in staging.

---

### 10. References

- Upstash Pricing – `https://upstash.com/pricing`
- Upstash Redis Documentation – `https://docs.upstash.com/redis`
- BullMQ with Upstash – `https://docs.bullmq.io/guide/connections`
