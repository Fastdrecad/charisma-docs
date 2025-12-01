## Backend Test Workflow

---

## 1. Overview

The backend test suite is separated into **unit tests** and **integration tests** to provide reliable coverage with fast feedback during development. This document describes how to run each test type, how the database is managed for integration tests, and how tests are used in pre-commit hooks and CI/CD.

---

## 2. Test Scripts

### 2.1 Unit tests (fast, no database required)

```bash
npm run test:unit
```

- Runs all unit tests (services, validation, completion tests).
- Uses mocks and does not require a database.
- Fast execution (typically ~2–3 seconds).
- Runs automatically during pre-commit hooks.

### 2.2 Integration tests (slower, database required)

```bash
npm run test:integration
```

- Runs all integration tests (API endpoints, database operations).
- Requires a PostgreSQL test database to be running.
- Slower execution (typically ~5–10 seconds).
- Must be run manually after database setup in local development.

### 2.3 Test database management

```bash
# Start test database and run migrations
npm run test:setup-db

# Stop and clean up test database
npm run test:teardown-db
```

---

## 3. Integration Test Workflow

To run integration tests locally, follow this sequence:

1. **Set up the test database:**

   ```bash
   npm run test:setup-db
   ```

2. **Run integration tests:**

   ```bash
   npm run test:integration
   ```

3. **Clean up (optional in local development):**

   ```bash
   npm run test:teardown-db
   ```

---

## 4. Test File Naming Conventions

- Unit tests: `*.service.test.ts`, `*.validation.test.ts`, `*.completion.test.ts`.
- Integration tests: `*.integration.test.ts`.

These conventions help distinguish test types and keep the suite organized.

---

## 5. Pre-commit Behavior

- Only unit tests run during pre-commit hooks.
- Integration tests are excluded from pre-commit to avoid database dependencies and slow feedback.
- ESLint and Prettier run on all staged files as part of the pre-commit process.

---

## 6. CI/CD Pipeline Workflow

In CI/CD environments, the recommended test workflow is:

1. Run unit tests.
2. Set up the test database.
3. Run integration tests.
4. Tear down or clean up the test database.

This ensures both fast feedback (unit tests) and full end-to-end coverage (integration tests) before changes are merged into the main branch.
