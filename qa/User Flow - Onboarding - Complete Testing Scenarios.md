# QA Test Cases - Complete List

---

## 📋 Complete QA Test Plan

### **🎯 CORE FLOWS (Must Test)**

```typescript
// ========================================
// 1. FRESH INSTALL - New User (Happy Path)
// ========================================
[CRITICAL] Fresh Install → Complete Onboarding
Steps:
1. [ ] Fresh install (no auth, no data)
2. [ ] Splash → Welcome screen appears
3. [ ] Tap "Get Started" → Carousel screen
4. [ ] Swipe through 3 carousel slides
5. [ ] Tap "Next" on last slide → Auth screen
6. [ ] Tap "Skip" → Auth screen
7.1 [ ] Tap "Continue with Google" → Auth success
7.2 [ ] Tap "Continue with Apple" → Auth success
8. [ ] Redirected to Survey screen (Question 1/5)
9. [ ] Complete all 5 questions
10. [ ] See "Thanks" screen
11. [ ] Tap "Let's Go" → Main app (Openers)

Expected: Smooth flow, no crashes, ~90 seconds total
Log checkpoints:
- "Not signed in → Welcome screen"
- "Auth success → Survey/1"
- "Survey complete → Thanks"
- "Onboarding complete → Main app"

// ========================================
// 2. RETURNING USER - Onboarding Complete
// ========================================
[CRITICAL] App Relaunch → Skip Onboarding
Steps:
1. [ ] User completed onboarding in previous session
2. [ ] Close app completely
3. [ ] Reopen app

Expected: Splash → Main app (Openers) directly
Log: "Onboarding complete → Main app"
Time: < 3 seconds

// ========================================
// 3. MID-ONBOARDING - User Exits During Survey
// ========================================
[HIGH] Resume Survey After App Close
Steps:
1. [ ] User starts survey (answers 2/5 questions)
2. [ ] Close app (don't complete survey)
3. [ ] Reopen app

Expected: Splash → Resume at Survey Question 1 (with previous answers preserved)
Log: "Survey in progress → Resume survey"

// ========================================
// 4. AUTHENTICATED BUT NO SURVEY - Auth Completed, App Closed
// ========================================
[HIGH] Auth Done, Survey Not Started
Steps:
1. [ ] User completes: Welcome → Carousel → Auth
2. [ ] Close app BEFORE starting survey
3. [ ] Reopen app

Expected: Splash → Welcome screen (restart from beginning)
Log: "Signed in but no survey progress → Welcome screen"

// ========================================
// 5. SKIP CAROUSEL - User Taps "Skip"
// ========================================
[MEDIUM] Skip Carousel Flow
Steps:
1. [ ] Fresh install → Welcome → Carousel
2. [ ] Tap "Skip" on first carousel slide

Expected:
- If not authenticated: → Auth screen
- If authenticated: → Survey screen
Log: "Carousel skipped → [Auth/Survey]"
```

---

### **🔀 NAVIGATION FLOWS**

```typescript
// ========================================
// 6. BACK NAVIGATION - User Presses Back
// ========================================
[MEDIUM] Back Button Behavior
Steps:
1. [ ] Fresh install → Welcome → Carousel → Auth
2. [ ] On Auth screen, press device back button

Expected: Go back to Carousel (not Welcome)
Note: Welcome should be replace(), not push()

// ========================================
// 7. CAROUSEL SWIPE - All Slides
// ========================================
[LOW] Carousel Interaction
Steps:
1. [ ] Welcome → Carousel
2. [ ] Swipe left through all 2 slides
3. [ ] Verify "Next" button on each slide
4. [ ] Verify "Skip" button on each slide
5. [ ] Last slide shows "Get Started" instead of "Next"

Expected: Smooth transitions, no crashes

// ========================================
// 8. AUTH ALREADY SIGNED IN - Skip Auth
// ========================================
[MEDIUM] User Already Authenticated at Carousel
Steps:
1. [ ] User authenticated (from previous session)
2. [ ] Open app → Welcome → Carousel
3. [ ] Tap "Next" on last carousel slide

Expected: Skip auth screen, go directly to Survey
Log: "Already authenticated → Skip auth, Survey/1"
```

---

### **🚨 ERROR HANDLING**

```typescript
// ========================================
// 9. AUTH FAILURE - OAuth Cancelled
// ========================================
[HIGH] User Cancels OAuth
Steps:
1. [ ] Welcome → Carousel → Auth
2. [ ] Tap "Continue with Google"
3. [ ] Close OAuth popup (cancel)

Expected: Stay on Auth screen, show error message
Log: "OAuth cancelled by user"

// ========================================
// 10. NETWORK FAILURE - No Internet During Profile Load
// ========================================
[HIGH] Offline Profile Load
Steps:
1. [ ] Turn off internet
2. [ ] Open app (user is signed in)

Expected:
- Show splash for max 10 seconds
- Timeout fallback kicks in
- Either continue with cached profile or show error
Log: "Profile load timeout - continuing anyway"

// ========================================
// 11. API ERROR - Backend Returns 500
// ========================================
[MEDIUM] API Failure Handling
Steps:
1. [ ] Kill backend server
2. [ ] Open app (user is signed in)

Expected: Graceful error handling, don't crash
Log: "Error loading profile"

// ========================================
// 12. SURVEY VALIDATION - Skip Required Question
// ========================================
[MEDIUM] Required Fields Validation
Steps:
1. [ ] Start survey
2. [ ] Try to skip Question 1 (Gender - required)

Expected: Cannot proceed without answering
UI: Show validation message

// ========================================
// 13. SURVEY INCOMPLETE - Exit Mid-Survey
// ========================================
[MEDIUM] Partial Survey Completion
Steps:
1. [ ] Answer questions 1-3 of 5
2. [ ] Close app
3. [ ] Reopen app

Expected: Resume at Question 1, but answers 1-3 are preserved
Log: "Survey in progress → Resume survey (answersCount: 3)"
```

---

### **⚡ PERFORMANCE**

```typescript
// ========================================
// 14. SPLASH SCREEN DURATION
// ========================================
[CRITICAL] Splash Screen Timing
Test:
1. [ ] Fresh install: Splash visible for 2-3 seconds ✅
2. [ ] Returning user: Splash visible for < 3 seconds ✅
3. [ ] Profile load timeout: Max 10 seconds ⚠️

Expected: Splash never visible for more than 10 seconds

// ========================================
// 15. NAVIGATION SPEED
// ========================================
[MEDIUM] Screen Transition Performance
Test:
1. [ ] Welcome → Carousel: < 300ms
2. [ ] Carousel → Auth: < 300ms
3. [ ] Auth → Survey: < 500ms (profile load)
4. [ ] Survey → Thanks: < 300ms
5. [ ] Thanks → App: < 500ms

Expected: No slow transitions (> 1 second)
Log: Check for "⚠️ Slow navigation detected" warnings
```

---

### **📱 DEVICE SCENARIOS**

```typescript
// ========================================
// 16. APP BACKGROUNDING - iOS/Android
// ========================================
[HIGH] Background/Foreground Transition
Steps:
1. [ ] Open app → Start survey (Question 2/5)
2. [ ] Press home button (background app)
3. [ ] Wait 5 seconds
4. [ ] Reopen app

Expected: Resume at same screen (Question 2), no state loss
Log: No router resets

// ========================================
// 17. MEMORY PRESSURE - Low Memory Kill
// ========================================
[LOW] App Killed by OS
Steps:
1. [ ] Start survey (Question 3/5)
2. [ ] Force kill app (simulate low memory)
3. [ ] Reopen app

Expected: Resume at Question 1, but answers 1-3 preserved
Note: Depends on AsyncStorage persistence

// ========================================
// 18. ORIENTATION CHANGE - Tablet Support
// ========================================
[LOW] Portrait/Landscape Rotation
Steps:
1. [ ] Open app on iPad
2. [ ] Rotate device during Welcome screen
3. [ ] Rotate during Carousel
4. [ ] Rotate during Survey

Expected: Layout adapts, no crashes, no state loss
```

---

### **🔐 AUTH EDGE CASES**

```typescript
// ========================================
// 19. MULTIPLE ACCOUNTS - Switch Account
// ========================================
[MEDIUM] User Switches Google Account
Steps:
1. [ ] Sign in with account A
2. [ ] Complete onboarding
3. [ ] Sign out
4. [ ] Sign in with account B

Expected: New onboarding flow for account B
Log: "New user detected → Welcome screen"

// ========================================
// 20. SESSION EXPIRY - Token Refresh
// ========================================
[LOW] Auth Token Expires
Steps:
1. [ ] Sign in
2. [ ] Wait for token to expire (manual test or mock)
3. [ ] Try to complete survey

Expected: Clerk auto-refreshes token, no user interruption
Log: "Token retrieved successfully"

// ========================================
// 21. SIGN OUT - Clear State
// ========================================
[HIGH] User Signs Out from Settings
Steps:
1. [ ] Complete onboarding → Main app
2. [ ] Go to Settings → Sign Out
3. [ ] Confirm sign out

Expected: Return to Welcome screen, all data cleared
Log: "User signed out → Welcome screen"
```

---

## 📊 QA Checklist Summary

### **Priority Levels:**

| Priority        | Count | Description                          |
| --------------- | ----- | ------------------------------------ |
| 🔴 **CRITICAL** | 3     | Must pass before release             |
| 🟠 **HIGH**     | 6     | Should pass, block release if broken |
| 🟡 **MEDIUM**   | 8     | Nice to have, fix before production  |
| 🟢 **LOW**      | 4     | Edge cases, can defer to v1.1        |

### **Test Coverage:**

```
Core Flows:        5 tests  (✅ Must pass)
Navigation:        3 tests  (✅ Must pass)
Error Handling:    5 tests  (⚠️ Should pass)
Performance:       2 tests  (⚠️ Monitor)
Device Scenarios:  3 tests  (⚠️ Nice to have)
Auth Edge Cases:   3 tests  (⚠️ Nice to have)
─────────────────────────────
Total:            21 tests
```

---

## 🎯 Minimal MVP Test List (If Short on Time)

Ako nemaš vremena za sve 21 testove, **ovo je minimum**:

```typescript
// ✅ MUST TEST (5 tests - 15 minutes)

1. [ ] Fresh install → Complete onboarding (happy path)
2. [ ] Returning user → Skip to app
3. [ ] Mid-survey → Resume survey
4. [ ] Auth cancelled → Stay on auth screen
5. [ ] Sign out → Return to welcome

// IF time allows, add:
6. [ ] Carousel skip
7. [ ] Back navigation
8. [ ] App backgrounding
```

---

## 🚀 Next Steps

**Da li želiš:**

1. ✅ **Implementirati minimal change** (vratiti stari kod, promeniti 1 liniju)
2. ✅ **Testirati core 5 scenarija** (happy path)
3. ✅ **Zatim dodati ostale testove** (edge cases)

**Ili želiš da prvo završimo implementaciju pa onda QA?** 🎯
