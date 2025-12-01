## Notification System – Production Verification Guide

This guide describes how to verify that the notification system is reliable and safe to run in production.

---

## 1. Automated Tests

### 1.1 Unit tests

```bash
# Test Notification Service
npm test -- notification.service.test.ts

# Test Daily Tips Job
npm test -- daily-tips.job.test.ts
```

**Coverage:**

- ✅ Token validation (valid/invalid Expo tokens)
- ✅ Single notification sending
- ✅ Bulk sending with filtering
- ✅ Error handling (API errors, network errors)
- ✅ Optional fields (data, sound, badge)
- ✅ Job logic (query users, pick tip, send)

### 1.2 Integration tests

```bash
# End-to-end with real database
npm run test:setup-db
npm run test:integration -- notification
npm run test:teardown-db
```

---

## 2. Manual Testing

### 2.1 Test push token registration

**Step 1: Register a token on the backend**

```bash
# POST /api/v1/users/me/push-token
curl -X POST http://localhost:8080/api/v1/users/me/push-token \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "pushToken": "ExponentPushToken[YOUR_TEST_TOKEN]"
  }'
```

Check in the database:

```sql
SELECT id, email, pushToken, pushTokenUpdatedAt
FROM users
WHERE pushToken IS NOT NULL;
```

Expected:

- Token persisted in the database.
- `pushTokenUpdatedAt` set.
- Response: `{ success: true, data: { user: {...} } }`.

### 2.2 Test sending a notification

Directly invoking `NotificationService`:

```typescript
import { NotificationService } from './modules/notification/notification.service.js';

const result = await NotificationService.sendNotification({
  pushToken: 'ExponentPushToken[YOUR_TOKEN]',
  title: '🧪 Test Notification',
  body: 'This is a test notification from production verification',
  data: { screen: '/(app)/(tabs)/openers' },
});

console.log('Sent:', result); // Should be true
```

Verify:

- Notification arrives on the device.
- Deep link works (tap navigates to the expected screen).
- Sound plays (if enabled).
- Badge updates (if configured).

### 2.3 Test the Daily Tips job

Manual job run:

```bash
# Use existing script
npm run test:notifications

# Or call the function directly
tsx -e "
import { sendDailyTips } from './src/jobs/daily-tips.job.js';
sendDailyTips().then(() => process.exit(0)).catch(err => { console.error(err); process.exit(1); });
"
```

Check logs:

```text
"Starting daily tips job"
"Daily tips sent successfully" with userCount > 0
```

Check in the database:

```sql
-- How many users have a token
SELECT COUNT(*) FROM users
WHERE pushToken IS NOT NULL AND deletedAt IS NULL;

-- Last token update
SELECT MAX(pushTokenUpdatedAt) FROM users;
```

---

## 3. Monitoring and Alerting

### 3.1 Key metrics to track

In the Pino logger, check:

```typescript
// Success rate
logger.info({
  message: 'Bulk push notifications sent',
  count: 100,
  successCount: 95, // <-- Check success rate
});

// Error rate
logger.error({
  message: 'Error sending push notification',
  error: '...',
});
```

Critical metrics:

- Success rate: target > 90% (over 9/10 notifications sent successfully).
- Invalid token rate: target < 5% (some invalid tokens are normal).
- Job execution: once per day at the scheduled time (for example, 10:00 AM).
- Error count: 0 critical errors in the last 24 hours (investigate any errors).

### 3.2 Recommended alerts

Add the following logic to your monitoring system (Sentry/Datadog/New Relic, etc.):

```javascript
// Alert 1: Daily job failed
if (error.message.includes('Error sending daily tips')) {
  alert('CRITICAL: Daily tips job failed');
}

// Alert 2: High failure rate
const successRate = successCount / totalCount;
if (successRate < 0.8) {
  alert('WARNING: Notification success rate below 80%');
}

// Alert 3: Job didn't run
if (!jobExecutedToday()) {
  alert('CRITICAL: Daily tips job did not run today');
}

// Alert 4: No users with tokens
if (userCount === 0 && expectedUsers > 0) {
  alert('WARNING: No users with push tokens found');
}
```

### 3.3 Health check endpoint

Example endpoint in `app.ts`:

```typescript
// GET /health/notifications
router.get('/health/notifications', async (req, res) => {
  const usersWithTokens = await prisma.user.count({
    where: {
      pushToken: { not: null },
      deletedAt: null,
    },
  });

  const lastJobRun = await getLastJobExecutionTime(); // Implement this

  res.json({
    success: true,
    data: {
      usersWithTokens,
      lastJobRun,
      expoApiStatus: 'operational', // Check Expo status API
    },
  });
});
```

---

## 4. Production Checklist

### 4.1 Pre-deployment

- [ ] **Tests pass:** `npm test` - all green
- [ ] **Token validation:** Invalid tokens are rejected
- [ ] **Error handling:** Network errors handled gracefully
- [ ] **Logging:** All actions logged (info/error)
- [ ] **Environment vars:** Expo SDK configured (if required)

### 4.2 Post-deployment

- [ ] **Token registration works:** Test user can register token
- [ ] **Manual send works:** Direct invocation of `sendNotification` works
- [ ] **Daily job works:** Job runs at 10:00 AM
- [ ] **Notifications arrive:** Users receive notifications
- [ ] **Deep links work:** Tap navigates user
- [ ] **Monitoring active:** Alerts set up

### 4.3 Daily checks

- [ ] **Daily job executed:** Check logs for "Daily tips sent successfully"
- [ ] **Success rate:** > 90% notifications sent successfully
- [ ] **Error log:** No critical notification-related errors
- [ ] **User count:** Users with tokens trending up or stable

---

## 5. Troubleshooting

### 5.1 Problem: users do not receive notifications

Diagnosis:

```bash
# 1. Check if user has a token in DB
SELECT id, email, pushToken, pushTokenUpdatedAt
FROM users
WHERE email = 'user@example.com';

# 2. Validate token format
# Token must start with "ExponentPushToken[..."
# And follow structure: ExponentPushToken[UUID-like-string]

# 3. Check Expo API status
curl https://exp.host/--/api/v2/status

# 4. Check error logs
# Look for: "Error sending push notification"
```

Solutions:

- Token expired – user must re-grant permission in the app.
- Invalid token format – check `Expo.isExpoPushToken()` validation.
- Expo API down – check Expo status page.
- Network error – rely on retry logic (see Recommended Improvements).

### 5.2 Problem: daily job does not run

Diagnosis:

```typescript
// Verify scheduler started in server.ts
import { startScheduledJobs } from './jobs/scheduler.js';
startScheduledJobs(); // <-- Must be called

// Check cron expression
cron.schedule('0 10 * * *', ...); // 10:00 AM every day
```

Solutions:

- Cron not started – add `startScheduledJobs()` to server startup.
- Incorrect cron expression – verify expression matches the intended schedule.
- Server timezone – verify server timezone vs expected time.

### 5.3 Problem: high failure rate (> 20%)

Diagnosis:

```typescript
// Inspect Expo tickets for details
const tickets = await expo.sendPushNotificationsAsync([...]);
tickets.forEach(ticket => {
  if (ticket.status === 'error') {
    console.error('Error:', ticket.message);
  }
});
```

Most common causes:

1. `DeviceNotRegistered`: token expired (device uninstalled the app).
   - Solution: periodically clean up invalid tokens from DB.
2. `MessageTooBig`: notification too large.
   - Solution: shorten title/body (for example, max ~100 characters).
3. `RateLimitExceeded`: too many requests.
   - Solution: add rate limiting or delays between batches.

### 5.4 Problem: deep links do not work

- Backend is sending `data: { screen: '/(app)/(tabs)/openers' }` (or similar).
- Issue is likely on the frontend – check `useNotificationListeners.ts` (or equivalent) for deep link handling.

---

## 6. Production Test Scenarios

### 6.1 Scenario: normal flow

```text
1. User registers a push token.
2. Backend persists the token.
3. Daily job runs at 10:00 AM.
4. Notifications are sent to all users with tokens.
5. Users receive notifications.
6. Tapping a notification navigates the user correctly.
```

Expected: all steps succeed.

### 6.2 Scenario: invalid token cleanup

```text
1. Job sends a notification with an expired token.
2. Expo returns "DeviceNotRegistered".
3. Backend logs the error.
4. Periodic cleanup removes invalid tokens.
```

Expected: invalid tokens are removed over time; success rate improves.

### 6.3 Scenario: bulk sending with small batch size

```text
1. 1000 users have tokens.
2. Job sends notifications in one call.
3. Expo SDK automatically chunks (> 100 notifications per chunk).
4. All notifications are sent successfully.
```

Expected: all 1000 notifications sent; success rate near 100%.

### 6.4 Scenario: network failure recovery

```text
1. Job attempts to send notifications.
2. Network error occurs when calling Expo API.
3. Job catches and logs the error.
4. Job runs again on the next scheduled execution.
```

Expected: error is logged; job does not crash; system recovers on subsequent runs.

---

## 7. Recommended Improvements

### 7.1 Short-term (pre-production)

1. **Retry logic za failed tokens**

   ```typescript
   // Add in daily-tips.job.ts
   const tickets = await NotificationService.sendBulkNotifications(...);
   const failedTokens = tickets
     .map((t, i) => ({ ticket: t, user: users[i] }))
     .filter(({ ticket }) => ticket.status !== 'ok');

   // Log failed for cleanup
   logger.warn({ message: 'Failed tokens', failedTokens });
   ```

2. **Token cleanup job**

   ```typescript
   // New function: cleanupInvalidTokens()
   // Run weekly
   // Deletes tokens invalid for >30 days
   ```

3. **Error tracking**

   ```typescript
   // Integrate Sentry for error tracking
   import * as Sentry from '@sentry/node';

   if (error) {
     Sentry.captureException(error, {
       tags: { feature: 'notifications' },
     });
   }
   ```

### 7.2 Long-term

1. **Analytics integration** - track delivery rate, open rate
2. **A/B testing** - test different notification types
3. **Personalization** - per-user customized notifications
4. **Rate limiting** - prevent spam

---

## 8. Testing Commands – Quick Reference

```bash
# Unit tests
npm test -- notification.service.test.ts
npm test -- daily-tips.job.test.ts

# Integration tests (requires test DB)
npm run test:setup-db
npm run test:integration -- notification
npm run test:teardown-db

# Manual job test
npm run test:notifications

# Type check
npm run type-check

# Lint
npm run lint
```

---

**Last updated:** 2025-01-27  
**Status:** Production-ready with recommended improvements
