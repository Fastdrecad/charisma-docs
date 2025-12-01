## Phase Detection System – Developer Guide

**Last Updated:** 2025-01-XX  
**Status:** MVP implementation in production  
**Priority:** High (latency and cost-sensitive component)

---

### 1. Overview

The Phase Detection system analyzes conversation messages and classifies the interaction into one of four phases.
The computed phase is used by downstream services (for example, suggestions, RAG) to adapt behavior to the user’s current context.

Phase definitions:

- **Phase 1 (Hook):** Initial engagement (roughly 1–3 messages).
- **Phase 2 (Bridge):** Personal exchange and discovery (roughly 4–8 messages).
- **Phase 3 (Pivot):** Future planning or escalation (roughly 9–12 messages).
- **Phase 4 (Close):** Number exchange or scheduling (roughly 13+ messages).

Performance context:

- Previous version: ~5.5s latency, 100% dependent on Gemini.
- Current version (MVP): ~150–250 ms latency for the majority of requests.
- Design: hybrid approach combining rule engine, caching, and asynchronous Gemini refinement.

---

### 2. Quick Start

#### 2.1 Prerequisites

- Docker (or equivalent) running Redis and PostgreSQL (`npm run dev:db`).
- Environment variables: `GEMINI_API_KEY`, `REDIS_URL` (plus standard backend env).

#### 2.2 Running the service and tests

```bash
# Start backend
npm run dev:local

# Run specific phase detection tests
npm test -- phase-detection

# Check metrics (locally, if Prometheus endpoint is enabled)
curl http://localhost:8080/metrics | grep phase_
```

---

### 3. Architecture and Workflow

The system uses a **complexity-based routing** architecture to decide whether a fast rule-based evaluation is sufficient or whether Gemini should be invoked.

#### 3.1 Decision flow

1. **Incoming request** – User requests a suggestion or phase update.
2. **Fingerprint cache** – A SHA256 hash of the full conversation history is checked:
   - Cache hit: return cached phase immediately.
   - Cache miss: continue to processing.
3. **Complexity calculation** – Analyze message count, sentiment, keyword patterns, escalation signals, and explicit user requests.
4. **Routing decision** (using `complexityThreshold`, e.g. 0.6):
   - **Fast path (score < threshold):** run rule engine only; return result (target latency < 10 ms).
   - **Needs-AI path (score ≥ threshold):**
     - Run rule engine and return its result immediately (non-blocking).
     - Fire-and-forget a background Gemini job for refinement.
     - When Gemini finishes, update cache and notify clients via WebSocket.
5. **Async update** – The frontend listens for phase update events and refreshes the UI when the refined phase is available.

#### 3.2 Key components

- **Rule engine** – Fast, pattern-based logic (regex, keyword counting) that provides a baseline phase classification.
- **Complexity scorer** – Computes a scalar score indicating whether Gemini is needed.
- **Fingerprint cache** – Uses a 12-character SHA256 hash of the entire conversation for precise cache keys.
- **WebSocket broadcaster** – Pushes `session_phase_update` events from background jobs to connected clients.

---

### 4. Project Structure

Location: `backend/src/modules/sessions/services/phase-detection/`

```text
├── config/
│   ├── index.ts          # Loader with Zod validation
│   └── weights.json      # Scoring weights & thresholds
├── complexity/           # Scorer Logic
│   ├── index.ts          # Main calculator & routing logic
│   ├── message-count.score.ts
│   ├── keyword.score.ts
│   ├── sentiment.score.ts
│   ├── explicit-request.score.ts
│   └── escalation.score.ts
├── rules/                # Fast-path Logic
│   ├── index.ts          # Orchestrator
│   ├── phase-1.rules.ts  # Hook rules
│   ├── phase-2.rules.ts  # Bridge rules
│   ├── phase-3.rules.ts  # Pivot rules
│   └── phase-4.rules.ts  # Close rules
├── metrics.ts            # Prometheus metric definitions
└── phase-detection.service.ts  # Main entry point
```

---

### 5. Core Logic and Implementation

#### 5.1 Fingerprint caching

The cache key is derived from the full conversation content (author + text), not just message count, to ensure correctness.

```typescript
const conversationFingerprint = messages
  .map((m) => `${m.author}:${m.text}`)
  .join("\n");
const hash = createHash("sha256")
  .update(conversationFingerprint)
  .digest("hex")
  .slice(0, 12);
const cacheKey = `phase:${sessionId}:${hash}`;
```

#### 5.2 Adaptive windows (MVP)

To minimize work per request, different phases use different recent-message windows:

- Phase 1 and 4: last 3 messages.
- Phase 2: last 5 messages.
- Phase 3: last 8 messages (requires more context).

These values are part of the MVP design and should be tuned carefully if changed.

#### 5.3 Complexity scoring

The `needsAI` flag is driven by a weighted score in the \[0.0, 1.0\] range.
Weights (see `weights.json`) typically include:

- Explicit requests (for example, “send me your number”) – high impact; often sufficient alone to require AI.
- Sentiment – negative sentiment raises complexity to avoid poor advice.
- Message count, keyword patterns, and escalation – remaining weight.

#### 5.4 Background Gemini processing

When `needsAI` is true, the user is not blocked on Gemini; the system:

```typescript
// 1. Get fast result
const fastResult = await useRuleEngine(messages);

// 2. Trigger background refinement
fireAndForget(async () => {
  const geminiResult = await useGemini(messages);
  updateCache(geminiResult);
  broadcastPhaseUpdate(sessionId, geminiResult); // WebSocket
});

// 3. Return fast result immediately
return fastResult;
```

---

### 6. Configuration

Configuration is defined in `config/weights.json` and validated at runtime using Zod to prevent invalid configurations (for example, weights not summing to ~1.0 or thresholds outside \[0, 1\]).

Key file:

- `backend/src/modules/sessions/services/phase-detection/config/weights.json`

```json
{
  "scoreWeights": {
    "messageCount": 0.15,
    "keywords": 0.15,
    "sentiment": 0.25,
    "explicitRequest": 0.3,
    "escalation": 0.15
  },
  "complexityThreshold": 0.6
}
```

Notes:

- In development, configuration can be reloaded via helper functions (for example, `reloadConfig()`), if exposed.
- Any production changes to weights or thresholds must be tested against the golden dataset.

---

### 7. Testing Strategy

We rely on three layers of tests (run via `npm test`):

1. **Unit tests**
   - Validate individual complexity scorers (for example, explicit request detection).
   - Validate rule engine decisions and confidence scores.
2. **Golden dataset tests** (for example, `golden-dataset.test.ts`)
   - Use real conversation logs with known ground-truth phases.
   - Regression check: ensure optimizations do not reduce accuracy below agreed thresholds (for example, 80%).
3. **Integration tests**
   - Exercise the path from API call through rule engine, background Gemini job, cache, and WebSocket updates.

Any change to scoring or routing logic should be accompanied by re-running golden dataset tests and reviewing outcomes.

---

### 8. Monitoring and Metrics

Prometheus metrics are used to monitor latency, routing decisions, and cache behavior:

| Metric name                     | Description                  | Example alert/SLO        |
| ------------------------------- | ---------------------------- | ------------------------ |
| `phase_rule_engine_latency_ms`  | Latency of the fast path     | p95 > 500 ms             |
| `phase_gemini_latency_ms`       | Latency of background Gemini | monitored, not SLO-bound |
| `phase_cache_hits_total`        | Cache hits/misses            | hit rate < 50%           |
| `phase_routing_decisions_total` | Counts of fast vs `needsAI`  | `needsAI` ratio > 30%    |

These metrics should be visible in dashboards and used to tune thresholds and weights.

---

### 9. Troubleshooting

#### 9.1 High latency on fast path (> 500 ms)

Possible causes:

- Redis connectivity problems or timeouts.
- Rule engine regex patterns causing catastrophic backtracking.

Actions:

- Inspect `phase_rule_engine_latency_ms` histograms.
- Review recent changes under `rules/` for problematic expressions.
- Verify Redis health and network latency.

#### 9.2 Frontend not updating after background processing

Possible causes:

- WebSocket subscription failure.
- Redis pub/sub or broadcaster errors.

Actions:

- Check logs for messages like `Failed to broadcast phase update`.
- Confirm the client is listening for `session_phase_update` events for the correct session.

#### 9.3 Configuration validation failures on startup

Possible causes:

- `weights.json` weights do not sum to approximately 1.0.
- Threshold values outside the \[0, 1\] range.

Actions:

- Fix the JSON config and ensure it passes Zod validation in `config/index.ts`.

#### 9.4 Excessive Gemini calls (high cost)

Possible causes:

- `complexityThreshold` set too low, routing too many requests into `needsAI`.
- Overly sensitive scorers (for example, keyword scorer) with high weights.

Actions:

- Inspect `phase_routing_decisions_total` to understand the fast vs `needsAI` ratio.
- Increase `complexityThreshold` (for example, from 0.6 to 0.7).
- Reduce weights for scorers that are over-firing.
