## Backend Deployment Guide

**Status:** Active  
**Purpose:** End-to-end deployment and infrastructure guide for the backend service.

---

## 1. Overview

This document describes how to deploy the backend service (including modules such as Sessions, Magic Opener, Webhooks, etc.) across environments.

---

## 2. Infrastructure (Current Stack)

### 2.1 Core services

1. **Backend Hosting**: Railway (Node.js app)
2. **Database**: PostgreSQL (Railway / Neon)
3. **Redis**: Upstash Redis (BullMQ queues, rate limiting, cache, WS pub/sub)
4. **Object Storage**: Cloudflare R2 (screenshots, assets)
5. **Secrets Management**: Doppler (production), `.env.local` (development)

---

## 3. Environment Variables

- **Source of truth:** `.env.example` + `backend/src/core/env/index.ts`
- All env vars are validated at startup via Zod (`envSchema`).

**Typical categories:**

- **Core**
  - `DATABASE_URL` – PostgreSQL connection string
  - `REDIS_URL` – Redis connection string
  - `PORT` – Backend HTTP port (default 8080)
- **Auth**
  - `CLERK_SECRET_KEY` – Clerk authentication
  - `CLERK_WEBHOOK_SECRET` – Clerk webhook verification
- **Storage**
  - `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `R2_ENDPOINT`
- **AI Providers**
  - `OPENAI_API_KEY`, `GEMINI_API_KEY`, `ANTHROPIC_API_KEY` (optional)
  - `AI_PROVIDER_STRATEGY_PRIMARY`, `AI_PROVIDER_STRATEGY_FALLBACKS`
- **Secrets Management**
  - `DOPPLER_TOKEN` – Doppler service token (production)

---

## 4. Deployment Steps

### 4.1 Database migrations

```bash
# Local development
npm run db:migrate

# Production (Railway)
npm run db:migrate:deploy
```

### 4.2 Environment setup

**Local development:**

- Create `.env.local` (gitignored) based on `.env.example`.
- Run local dependencies (DB/Redis) as per `ONBOARDING.md` (e.g. Docker compose, dev scripts).

**Staging / Production:**

- Use **Doppler** as the single source of truth for secrets.
- In Railway, only set `DOPPLER_TOKEN`; all other secrets are injected by Doppler.

### 4.3 Build and deploy

```bash
# Build
npm run build

# Deploy (Railway handles this automatically)
```

---

## 5. Hosting Configuration (Railway)

**Build command:**

```bash
npm install && npm run build
```

**Start command:**

```bash
npm start
```

**Environment:**

- Set `DOPPLER_TOKEN` in Railway env vars.
- All other secrets managed by Doppler project/environment.

---

## 6. Cloudflare R2

**Bucket Setup:**

1. Create bucket in Cloudflare dashboard
2. Generate API tokens
3. Set CORS policy (if needed)

**CORS Policy:**

```json
[
  {
    "AllowedOrigins": ["https://app.charismaai.com"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedHeaders": ["*"],
    "MaxAgeSeconds": 3600
  }
]
```

## 7. Upstash Redis

**Setup:**

1. Create Redis database in Upstash
2. Copy connection string
3. Set in environment variables

**Configuration:**

- Free tier: 10K commands/day
- Sufficient for MVP

---

## 8. Monitoring

### 8.1 Logs

- Use Pino logger (`backend/src/core/logger/`) for structured JSON logs.
- View logs in Railway dashboard (production/staging).

### 8.2 Metrics (Sessions-related)

- Session creation rate
- OCR processing time
- AI suggestion generation latency
- Cache hit/miss ratio
- Error rates (HTTP 5xx, AI provider errors)

---

## 9. Rollback Strategy

### 9.1 Database

- Prefer **forward-only migrations**; use new migrations to fix issues.
- In exceptional cases, you can manually resolve migrations via Prisma CLI.

### 9.2 Application

- Railway supports automatic rollback on deployment failure.
- For manual rollback, redeploy a previously known-good build.

---

## 10. Cost Overview (High Level)

- Cloudflare R2: object storage (screenshots, prompts) – low volume.
- Upstash Redis: free/low tier sufficient for queues + rate limiting for MVP.
- Railway: primary recurring cost (app + database).
- AI Providers: depends on traffic (OpenAI / Gemini / Anthropic usage).

---

## 11. Security

### 11.1 Secrets management

- **Production:** Doppler (only `DOPPLER_TOKEN` stored in hosting provider).
- **Development:** `.env.local` (gitignored); **never** commit secrets.

### 11.2 Authentication

- Clerk JWT tokens for all HTTP endpoints.
- WebSocket authentication via `authenticate` action (token in payload).

### 11.3 File upload

- Pre-signed URLs with short expiry (R2).
- Direct R2 upload from client (no backend proxy for file body).
- Content-type and size validation on both client and server.

---

## 12. References

- `docs/backend/ONBOARDING.md` – local setup, env, core modules
- `docs/backend/BACKEND_TECH_STACK_COMPLETE.md` – full backend stack details
- `docs/backend/sessions/SESSIONS_API_CONTRACT.md` – Sessions API & WebSocket contract

---

## 13. Notes

- Vector index created manually (Prisma limitation)
- Test database on port 5433 (separate from dev)
- Use Doppler for production secrets only
