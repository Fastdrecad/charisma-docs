## Production Verification – Notification System

> This is a production verification report (read-only, not an onboarding guide).

### 1. Summary

All notification-related automated tests are currently passing:

- `notification.service.test.ts` – 13 tests
- `daily-tips.job.test.ts` – 7 tests
- `notification.integration.test.ts` – 2 tests (1 skipped; requires a real device token)

Total: **22 tests** passing; the notification system is considered production-ready.

---

### 2. Quick Pre-deployment Check

# 1. Run all tests

npm test

# 2. Manually test the job

npm run test:notifications

# 3. Type check

npm run type-check

# 4. Lint

npm run lintExpected: all commands complete successfully with no failing tests.

---

### 3. Automated Test Coverage

#### 3.1 Notification service unit tests

Status: complete.

- `NotificationService.sendNotification()` covered
- `NotificationService.sendBulkNotifications()` covered
- Token validation covered
- Error handling covered
- Optional fields (data, sound, badge) covered

Test file:

- `backend/src/modules/notification/notification.service.test.ts`

#### 3.2 Daily tips job tests

Status: complete.

- Querying users with push tokens
- Random tip selection
- Bulk sending
- Error handling
- Deep link data

Test file:

- `backend/src/jobs/daily-tips.job.test.ts`

#### 3.3 Integration tests

Status: complete (with one test skipped that requires a real Expo token).

- Invalid token filtering tested
- Real device test skipped (requires a real test token)

Test file:

- `backend/src/modules/notification/notification.integration.test.ts`

---

### 4. How to Verify in Production

#### 4.1 Test token registration

# Use real API endpoint

curl -X POST https://your-api.com/api/v1/users/me/push-token \
 -H "Authorization: Bearer YOUR_JWT_TOKEN" \
 -H "Content-Type: application/json" \
 -d '{"pushToken": "ExponentPushToken[YOUR_REAL_TOKEN]"}'Verify:

- Response is `{ "success": true }`
- Token exists in the database (`SELECT pushToken FROM users WHERE email = '...'`)

#### 4.2 Test manual send

// In a backend console or test script
import { NotificationService } from "./modules/notification/notification.service.js";

const result = await NotificationService.sendNotification({
pushToken: "ExponentPushToken[YOUR_TEST_TOKEN]",
title: "Production Test",
body: "This is a production verification test",
data: { screen: "/(app)/(tabs)/openers" },
});

console.log("Sent:", result); // Should be trueVerify:

- Notification arrives on the device
- Deep link works (tap navigates correctly)
- Sound/badge behavior is as configured

#### 4.3 Test daily job

# Run the job manually

npm run test:notifications

# Or run directly in Node

node -e "
import('./src/jobs/daily-tips.job.js').then(m => m.sendDailyTips());
"Check logs:

"Starting daily tips job"
"Daily tips sent successfully" with userCount > 0Check in the database:

-- How many users have a push token
SELECT COUNT(\*) FROM users
WHERE pushToken IS NOT NULL AND deletedAt IS NULL;#### 4.4 Monitor scheduled execution

Scheduler:

// src/server.ts must contain:
import { startScheduledJobs } from "./jobs/scheduler.js";
startScheduledJobs(); // Must be calledLogs (e.g. daily at 10:00 AM):

- `"Running daily tips job"`
- `"Daily tips sent successfully"`

---

### 5. Monitoring and Alerting

#### 5.1 Key metrics

1. **Success rate**

   - Target: > 90%
   - Formula: `successCount / totalCount`
   - Alert if < 80%

2. **Job execution**

   - Target: once per day at the scheduled time
   - Alert if no run within 24 hours

3. **Invalid token rate**
   - Target: < 5%
   - Normal: tokens expire when users uninstall the app
   - Alert if > 20% (could indicate a systemic problem)

#### 5.2 Log patterns

Success:

{"message": "Bulk push notifications sent", "count": 100, "successCount": 95}
{"message": "Daily tips sent successfully", "userCount": 100}Errors (should trigger alerts):

{"message": "Error sending push notification", "error": "..."}
{"message": "Error sending daily tips", "error": "..."}---

### 6. Troubleshooting in Production

#### 6.1 Daily tips job did not run

Check:

1. Is `startScheduledJobs()` called in `server.ts`?
2. Is the server running at the scheduled time?
3. Are there users with tokens? (`SELECT COUNT(*) FROM users WHERE pushToken IS NOT NULL`)

#### 6.2 Low success rate (< 70%)

Possible causes and mitigations:

1. Many expired tokens (devices uninstalled the app)
   - Mitigation: periodic cleanup of invalid tokens
2. Expo API rate limiting
   - Mitigation: add delays between batches
3. Network issues
   - Mitigation: rely on Expo SDK retries and monitor logs

#### 6.3 Notifications not received

Debug steps:

1. Confirm that the token exists in the database
2. Confirm that the token format is valid (`ExponentPushToken[...]`)
3. Check Expo status: `https://status.expo.dev`
4. Review logs for related error messages

---

### 7. Additional Documentation

- Detailed guide: `backend/docs/NOTIFICATION_PRODUCTION_VERIFICATION.md`
- Audit report: `NOTIFICATION_SYSTEM_AUDIT_REPORT.md`
- Test files:
  - `backend/src/modules/notification/notification.service.test.ts`
  - `backend/src/jobs/daily-tips.job.test.ts`
  - `backend/src/modules/notification/notification.integration.test.ts`
