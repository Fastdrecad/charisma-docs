## 1. Overview

This document describes the current state and target design for **AI cost tracking, analytics, and model performance monitoring** in the backend.
It is intended as a reference for all backend and data engineers working on AI-related features.

Goals:

- Track token usage, latency, and cost per provider/model.
- Enforce safety limits (per-user and global budgets).
- Analyze usage and performance over time (dashboards, reports).
- Optimize model choice via A/B testing and automated promotion.

The sections below are organized as:

- Existing capabilities (what we have today).
- Gaps vs. 2025 industry practices.
- Proposed design and concrete implementation steps.

---

## 2. Current Capabilities

### 2.1 Cost tracking and limits

- `AICostLog` table persists:
  - `model`
  - `useCase`
  - `promptTokens`
  - `completionTokens`
  - `cost`
  - `createdAt`
- `cost-tracker.service.ts`:
  - Computes cost from token counts via `token-estimator.ts` and `MODEL_PRICES` map (with safe defaults for unknown models).
  - Enforces a **per-user daily cost limit** via `USER_DAILY_COST_LIMIT_USD` (throws `COST_LIMIT_EXCEEDED` and logs a warning).

### 2.2 Observability

- Logging includes:
  - Provider and model names.
  - Token counts (input/output).
  - Latency measurements for major AI calls.
- Multi-provider strategy is centralized (Gemini, OpenAI, Anthropic), with clear logging of fallbacks and retries.

### 2.3 Model usage

- AI usage is indirectly observable via:
  - `AICostLog` (per-call cost and tokens).
  - Domain tables such as `AIGeneratedOpener`, `Session`, `Message`, and interaction tables for engagement.

---

## 3. Gaps vs. Recommended Practices (2025)

The following capabilities are missing or partial:

- **Analytics and aggregation**

  - No dedicated service to aggregate costs, tokens, latency, and engagement into a consumable API.
  - No rollup tables for fast time-series queries (dashboards).

- **Budgeting and alerting**

  - Per-user daily limit exists.
  - No global / per-provider monthly budget thresholds or alerts (e.g., Slack notifications).

- **Structured logging**

  - Logging is mostly structured but not standardized for analytics (e.g. consistent `event` field, common schema).

- **Model A/B testing**

  - No systematic way to split traffic between models and measure which one performs better on user-centric metrics.

- **Automatic model promotion**
  - Model selection is static or manually tuned.
  - No feedback loop that promotes better-performing variants.

---

## 4. Analytics Service and API Design

### 4.1 Service responsibilities

Introduce an `ai-analytics.service.ts` with the following responsibilities:

- Aggregate **cost, tokens, and latency** from `AICostLog`.
- Expose **time-series** and **breakdown** metrics for dashboards and reports.
- Provide **provider/model comparison** views for decision making.

### 4.2 Proposed service API

```ts
// ai-analytics.service.ts (service interface)
export interface AIAnalyticsService {
  getCostStatsByModel(params: { from: Date; to: Date }): Promise<
    Array<{
      model: string;
      totalCostUsd: number;
      totalPromptTokens: number;
      totalCompletionTokens: number;
      callCount: number;
    }>
  >;

  getCostStatsByProvider(params: { from: Date; to: Date }): Promise<
    Array<{
      provider: string;
      totalCostUsd: number;
      callCount: number;
    }>
  >;

  getUsageOverTime(params: {
    from: Date;
    to: Date;
    interval: "day" | "week" | "month";
  }): Promise<
    Array<{
      bucketStart: Date;
      totalCostUsd: number;
      totalTokens: number;
      callCount: number;
    }>
  >;

  getTokenEfficiency(params: { model: string; from: Date; to: Date }): Promise<{
    model: string;
    promptTokens: number;
    completionTokens: number;
    efficiency: number; // completionTokens / promptTokens
  }>;

  getLatencyStats(params: { model?: string; from: Date; to: Date }): Promise<{
    p50Ms: number;
    p90Ms: number;
    p99Ms: number;
    avgMs: number;
    callCount: number;
  }>;
}
```

These methods should use:

- The `AICostLog` table as the primary source of truth.
- Optionally, a rollup table (see section 5) for pre-aggregated daily/monthly metrics.

### 4.3 HTTP endpoints (admin / internal)

Expose read-only analytics under an authenticated admin namespace, for example:

- `GET /admin/ai/cost/by-model`
- `GET /admin/ai/cost/by-provider`
- `GET /admin/ai/usage/over-time`
- `GET /admin/ai/latency`

Guidelines:

- Endpoints must be **authenticated** (admin JWT / API key).
- All responses should be **JSON**, designed for consumption by dashboards (internal admin UI, Grafana, etc.).
- Filter by `from` / `to` and possibly provider/model.

---

## 5. Cost Rollups and Background Jobs

### 5.1 Rollup table

Introduce a daily rollup table, for example:

- `ai_cost_rollup`:
  - `id` (primary key)
  - `date` (UTC date bucket)
  - `provider`
  - `model`
  - `totalCostUsd`
  - `totalPromptTokens`
  - `totalCompletionTokens`
  - `avgLatencyMs`
  - `callCount`

Benefits:

- Fast queries for charts (daily over the last N days).
- Lightweight analytics queries instead of scanning raw logs.

### 5.2 Rollup job

Implement a daily cron job (for example via BullMQ worker) that:

- Selects all `AICostLog` entries for the previous day.
- Aggregates per `(provider, model, date)`.
- Upserts into `ai_cost_rollup`.

Job requirements:

- Idempotent per date (safe to re-run).
- Instrumented with:
  - Start/end logs.
  - Duration.
  - Number of rows processed and upserted.

---

## 6. Budget Limits and Alerting

### 6.1 Existing behavior

- `USER_DAILY_COST_LIMIT_USD` is enforced per user in `cost-tracker.service.ts`.
- When exceeded:
  - A `COST_LIMIT_EXCEEDED` error is thrown.
  - A warning is logged.

### 6.2 Global / monthly budgets

Add support for **global and per-provider monthly budgets**, for example:

- Environment / secrets:
  - `AI_MAX_MONTHLY_COST_USD`
  - `AI_MAX_MONTHLY_COST_OPENAI_USD`
  - `AI_MAX_MONTHLY_COST_GEMINI_USD`

Implement a helper in an `ai-budget.service.ts`:

```ts
if (monthlyCost > config.maxMonthlyCost) {
  alertSlack("AI monthly cost limit reached for OpenAI provider", {
    monthlyCost,
    threshold: config.maxMonthlyCost,
  });
}
```

Alert channels:

- Slack webhook (recommended).
- Email to on-call or cost owner.
- Logging to a dedicated alert index.

Scheduling:

- Either part of the daily rollup job.
- Or a separate scheduled job that queries `AICostLog` / `ai_cost_rollup` for the current month.

---

## 7. Structured Logging and Correlation

### 7.1 Structured JSON logs

For AI-related calls, standardize on a common log schema:

- `event`: `"ai_request"` / `"ai_response"` / `"ai_error"`.
- `provider`, `model`.
- `latencyMs`.
- `costUsd`.
- `tokens.in`, `tokens.out`.
- `traceId`.
- `userId` (if available).

Example:

```ts
logger.info({
  event: "ai_request",
  provider,
  model,
  latencyMs,
  costUsd,
  tokens: { in: promptTokens, out: completionTokens },
  traceId,
  userId,
});
```

This makes logs easy to consume in Datadog, Grafana Loki, or ELK (Elasticsearch / OpenSearch).

### 7.2 Correlation IDs

Every inbound HTTP request / background job should:

- Generate or propagate a `traceId`.
- Attach it to the request/job context.
- Pass it through to all AI calls and logging.

Benefits:

- End-to-end tracing of a single user action or job.
- Faster debugging of high-cost or slow executions.

---

## 8. Model A/B Testing and Auto-Promotion

### 8.1 Goal

Systematically compare multiple models (or providers) on:

- User engagement metrics (likes, bookmarks, pins, conversions).
- Latency and reliability.
- Cost per “unit of value” (for example, per engaged suggestion).

### 8.2 Traffic splitting

Extend `ai-strategy.ts` so that, for specific use cases (for example, suggestions, openers), traffic is randomly split:

```ts
const roll = Math.random();
let model: string;
let abGroup: "A" | "B";

if (roll < 0.5) {
  model = "gpt-4o-mini";
  abGroup = "A";
} else {
  model = "gemini-2.5-flash";
  abGroup = "B";
}
```

Persist `abGroup` alongside generated content, for example:

- Add an `abGroup` field to `AIGeneratedOpener`.

### 8.3 Engagement scoring

Derive an engagement score per generated item:

```ts
const engagementScore = likes * 2 + bookmarks * 1.5 + pins * 3;
```

Inputs:

- Existing interaction tables:
  - `UserOpenerInteraction`
  - `UserMessageInteraction`

Analytics can then compute:

- Average engagement per `(model, abGroup)`.
- Cost per engagement unit:

```text
costPerScore = totalCostUsd / totalEngagementScore
```

### 8.4 A/B analytics endpoint

Expose an internal endpoint, for example `GET /admin/ai/ab-results`, returning:

```json
{
  "gpt-4o-mini": { "avgScore": 2.8, "costPerScore": 0.0006 },
  "gemini-2.5-flash": { "avgScore": 3.1, "costPerScore": 0.0004 }
}
```

Data sources:

- `AIGeneratedOpener` (model, abGroup).
- Interactions / engagement.
- `AICostLog` (cost, tokens).

### 8.5 Automatic model promotion

Once A/B results are stable over a defined window (for example, 7–14 days):

- If variant B has `avgScore` at least **10% higher** than baseline A, and `costPerScore` is equal or better, then:
  - Update `AI_PROVIDER_STRATEGY_PRIMARY` to the better variant.
  - Log this change and optionally send an alert (Slack/email).

This logic can live in:

- A dedicated `ai-model-governance.service.ts`.
- A scheduled job that periodically evaluates A/B results.

---

## 9. Summary and Next Steps

Current state:

- Token-level logging and `AICostLog` table are implemented.
- Per-user daily cost limits are enforced.
- Multi-provider AI strategy is in place with clear logging.

Recommended next steps:

1. Implement `ai-analytics.service.ts` and associated admin analytics endpoints.
2. Add `ai_cost_rollup` table and a daily rollup job.
3. Introduce global and per-provider monthly budgets with Slack/email alerts.
4. Standardize structured logging and correlation IDs across AI calls.
5. Implement model A/B testing and automated model promotion.

With these additions, the system will provide cost and performance monitoring comparable to modern AI-first engineering organizations.
