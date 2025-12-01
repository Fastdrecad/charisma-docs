## Charisma Documentation

This directory contains the technical documentation for the Charisma project.

---

## Documentation Index

### Core Product Flows

- **Authentication Flow:** [`client/AUTHENTICATION_FLOW.md`](./client/AUTHENTICATION_FLOW.md) – full auth + onboarding user journey.

### Backend

- **Backend Onboarding:** [`backend/ONBOARDING.md`](./backend/ONBOARDING.md) – global backend onboarding and architecture overview.
- **Backend Tech Stack:** [`backend/BACKEND_TECH_STACK_COMPLETE.md`](./backend/BACKEND_TECH_STACK_COMPLETE.md) – detailed stack and versions.
- **Controller Pattern Standard:** [`backend/CONTROLLER_PATTERN_STANDARD.md`](./backend/CONTROLLER_PATTERN_STANDARD.md) – controller conventions.
- **Deployment:** [`backend/DEPLOYMENT.md`](./backend/DEPLOYMENT.md) – deployment and verification.
- **Sessions Backend Onboarding:** [`backend/sessions/SESSIONS_BACKEND_ONBOARDING.md`](./backend/sessions/SESSIONS_BACKEND_ONBOARDING.md) – Sessions backend feature overview.
- **Sessions API Contract:** [`backend/sessions/SESSIONS_API_CONTRACT.md`](./backend/sessions/SESSIONS_API_CONTRACT.md) – REST and WebSocket contract for Sessions.
- **Sessions Testing Guide:** [`backend/sessions/TESTING.md`](./backend/sessions/TESTING.md) – Sessions testing strategy.
- **Magic Openers Backend Guide:** [`backend/magic/MAGIC_OPENERS_BACKEND_GUIDE.md`](./backend/magic/MAGIC_OPENERS_BACKEND_GUIDE.md) – Magic Openers feature backend.
- **Notifications Production Verification:** [`backend/PRODUCTION_VERIFICATION_SUMMARY.md`](./backend/PRODUCTION_VERIFICATION_SUMMARY.md) – notification system production verification.

### Frontend

- **Authentication Flow (client-facing):** [`client/AUTHENTICATION_FLOW.md`](./client/AUTHENTICATION_FLOW.md) – full auth + onboarding user journey from app perspective.

Quality assurance test cases for authentication and user flows:

- **NEW: Updated test cases for Welcome/Carousel/Survey flow**
- Fresh install complete onboarding (happy path)
- Returning user scenarios
- Mid-onboarding app close & resume
- Auth skip scenarios
- Network error handling
- Performance benchmarks

**Priority Levels:**

- 🔴 Critical (7 tests) - Must pass before launch
- 🟡 High (15 tests) - Should pass before production
- 🟢 Medium (13 tests) - Nice to have

---

## Quick Start Guides

### For Backend Developers

1. **Day 0: Product & flows (optional but recommended)**
   - Read: `client/AUTHENTICATION_FLOW.md` (core user journey)
2. **Day 1: Backend entrypoint**
   - Read: `../backend/README.md` (overview, scripts, environments)
   - Then: `../backend/DEVELOPMENT.md` (Neon dev branch vs Docker, `.env.local` vs `.env.docker`)
3. **Secrets & environments**
   - Read: `../backend/DOPPLER_SETUP.md` (how Doppler, `.env`, `.env.docker`, `.env.local` fit together)
4. **First‑day commands (copy/paste)**
   - Clone & install: `git clone … && cd backend && npm install`
   - Local dev (recommended): `cp .env.local.example .env.local` (if exists) → set `DATABASE_URL` (Neon dev) → `npm run dev:local`
   - Or full‑stack via Docker: `cp .env.docker.example .env.docker` → set Clerk + Doppler Service Token → `npm run dev:build`
5. **Quality & tests**
   - Before first PR: run `npm run lint`, `npm run type-check`, `npm run test`, `npm run build`
6. **Deeper backend docs (after you’re unblocked)**
   - Architecture & conventions: `backend/ONBOARDING.md`, `backend/BACKEND_TECH_STACK_COMPLETE.md`, `backend/CONTROLLER_PATTERN_STANDARD.md`
   - Feature deep‑dives: `backend/sessions/SESSIONS_BACKEND_ONBOARDING.md`, `backend/magic/MAGIC_OPENERS_BACKEND_GUIDE.md`, `backend/docs/NOTIFICATIONS.md`

### For Frontend Developers

1. **Authentication Flow** – `client/AUTHENTICATION_FLOW.md`
2. **Sessions Changes / Integration** – `client/SESSIONS_CHANGES.md` and `client/sessions-integration-best-practice.md`
3. **Client API README** – `client/src/lib/api/README.md`

### For QA Engineers

1. **QA Test Plan v1** – `docs/qa/qa-test-plan-v1.md`
2. **Onboarding Flow Scenarios** – `docs/qa/User Flow - Onboarding - Complete Testing Scenarios.md`
3. **Authentication Flow** – reference for expected behavior

---

## Documentation Status

| Area             | Key Doc                                           | Status      | Notes              |
| ---------------- | ------------------------------------------------- | ----------- | ------------------ |
| Product Flows    | `client/AUTHENTICATION_FLOW.md`                   | ✅ Updated  | v1.0 MVP flow      |
| Backend Intro    | `backend/ONBOARDING.md`                           | ✅ Updated  | Primary entrypoint |
| Backend Stack    | `backend/BACKEND_TECH_STACK_COMPLETE.md`          | ✅ Updated  | Check file         |
| Sessions Backend | `backend/sessions/SESSIONS_BACKEND_ONBOARDING.md` | ✅ Updated  | Check file         |
| Magic Openers    | `backend/magic/MAGIC_OPENERS_BACKEND_GUIDE.md`    | ✅ Updated  | Check file         |
| Client API       | `client/src/lib/api/README.md`                    | ✅ Complete | Check file         |

---

## Contributing

When adding new documentation:

1. Create a new `.md` file in this directory
2. Add an entry to this README with:
   - Brief description
   - Key topics covered
   - Status badge (✅ Complete / 🚧 In Progress / ❌ Outdated)
3. Follow the existing documentation structure and format
4. Include code references with line numbers where appropriate
5. Add version/date information for major updates

### Documentation Standards

- Use clear, descriptive headers
- Include code examples with syntax highlighting
- Add diagrams for complex flows (ASCII or Mermaid)
- Link to related documentation
- Mark deprecated content clearly with ⚠️ or ❌
- Update the index when making major changes

---

## Version History

### v1.0 (December 15, 2025) - MVP Launch

- **Major Update:** New user flow (Welcome → Carousel → Auth → Survey → Thanks → App)
- Auth moved after value proposition
- Survey reduced to 5 essential questions
- Added Welcome, Carousel, and Thanks screens

### v0.1 (Initial)

- Original authentication flow documentation
- Direct auth → onboarding flow
