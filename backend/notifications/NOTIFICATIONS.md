## Backend Notifications Guide

This document describes the backend implementation of push notifications (Expo), including the data model, API contracts, services, jobs, scripts, and testing.

---

## 1. Overview

- Transport: Expo push notifications (`expo-server-sdk`).
- Token source: mobile app registers an Expo push token per user.
- Storage: `users.pushToken`, `users.pushTokenUpdatedAt`.
- API: `POST /api/v1/users/me/push-token`.
- Service: `NotificationService` (single and bulk send).
- Jobs: `sendDailyTips` (example broadcast job).

---

## 2. Data Model

Prisma `User` fields related to notifications:

- `pushToken: string | null`
- `pushTokenUpdatedAt: Date | null`

These fields are set via the push-token API and read by notification jobs and services.

---

## 3. API

### 3.1 Register / update push token

- Route: `POST /api/v1/users/me/push-token`.
- Auth: Clerk (JWT) required.
- Body:

```json
{ "pushToken": "ExponentPushToken[xxxxx]" }
```

Responses:

- 200: `{ success: true, data: { user: { ... } } }`
- 400: `{ success: false, message: "Invalid push token" }`
- 401: `{ success: false, message: "Unauthorized" }`

Handler implementation: `modules/user/user.controller.ts` (`updatePushToken`).

---

## 4. Services

Main service: `modules/notification/notification.service.ts`.

- `NotificationService.sendNotification({ pushToken, title, body, sound?, badge?, data? })`
  - Validates with `Expo.isExpoPushToken`.
  - Returns `true` when the Expo ticket has `status === 'ok'`, otherwise `false`.
- `NotificationService.sendBulkNotifications([...])`
  - Filters invalid tokens.
  - Sends notifications in chunks.
  - Returns an array of Expo tickets.

Usage example:

```ts
import { NotificationService } from '@/modules/notification/notification.service';

await NotificationService.sendNotification({
  pushToken: 'ExponentPushToken[xxxx]',
  title: 'Hello',
  body: 'World',
  data: { screen: '/(app)/(tabs)/openers' },
});
```

---

---

## 5. Jobs

Job file: `jobs/daily-tips.job.ts`.

- Queries all users with non-null `pushToken`.
- Picks a random tip and broadcasts via `sendBulkNotifications`.

Run ad-hoc locally:

```bash
npm run test-daily-tips
```

---

## 6. Local Testing and Scripts

- Send a one-off test notification:

```bash
npm run test-notification -- ExponentPushToken[YOUR_TEST_TOKEN]
```

- Execute broadcast example (daily tips):

```bash
npm run test-daily-tips
```

Tests:

```bash
# Unit + integration (notifications included)
npm test

# Only integration tagged notification tests
npm run test:integration -- notification
```

---

## 7. Operational Notes

- Expo push does not require a server-side secret; validity is per-device token.
- Successes and failures are logged with `pino` for observability.
- Invalid tokens are ignored in bulk sends; individual sends return `false` when tickets indicate errors.

---

## 8. Troubleshooting

- “Invalid push token”: the token must be in Expo format `ExponentPushToken[xxx]`.
- No notifications received: ensure the client registered the token and it is stored on the user.
- Integration failures: verify device token freshness and that the app is built with Expo notifications enabled.

For production verification and monitoring details, see `backend/docs/NOTIFICATION_PRODUCTION_VERIFICATION.md`.
