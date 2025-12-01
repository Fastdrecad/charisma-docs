# 🔄 UPDATE - New User Flow (v1.0 - MVP)

## Changed Flow (October 30, 2025)

### OLD FLOW (Deprecated):

Splash → Auth → Onboarding → App

### NEW FLOW (Current):

Splash → Welcome → Carousel → Auth → Survey → Thanks → App

### Key Changes:

- ✅ Auth moved AFTER welcome/carousel (show value first)
- ✅ Onboarding renamed to "Survey" (5 questions)
- ✅ Added Welcome screen (3 features)
- ✅ Added Carousel (3 slides - value prop + social proof)
- ✅ Added Thanks screen (completion)

---

# 📋 Updated QA Test Plan

## 🎯 CORE FLOWS (Updated for New Flow)

### A. NEW USER (First Time User)

#### A1. Fresh Install → Complete Onboarding ✅ UPDATED

Steps:

1. Install app (fresh)
2. Open app
3. **See Welcome screen** (3 features) ← NEW
4. Tap "Get Started"
5. **See Carousel** (3 slides) ← NEW
6. Swipe through carousel or tap "Skip"
7. Tap "Next" on last slide → Auth screen
8. Tap "Continue with Google/Apple"
9. Complete OAuth authentication
10. **See Survey** (5 questions) ← RENAMED from "onboarding"
11. Answer all 5 questions
12. **See Thanks screen** ← NEW
13. Tap "Let's Go!"
14. See app (Openers screen)

Expected:

✅ Smooth flow: Welcome → Carousel → Auth → Survey → Thanks → App
✅ No auth screen flash before welcome
✅ Profile saved to backend
✅ onboardingCompleted = true
Time: ~2 minutes

Run mode: First run with `expo start --clear`; subsequent runs without `--clear`.

#### A2. Fresh Install → Exit During Survey ✅ UPDATED

Steps:

1. Install app
2. Complete: Welcome → Carousel → Auth
3. Start survey (answer 2/5 questions)
4. Force close app
5. Reopen app

Expected:

✅ Resume at Survey Question 1
✅ Previous answers (2/5) preserved in onboarding store
✅ Skip Welcome/Carousel (already saw them)

Run mode: Without `--clear` (warm start realism).

---

## 🎯 Updated Test Cases (Top Priority)

## 🔴 CRITICAL TESTS (Must Pass Before Launch)

### 1. Happy Path - Complete New User Flow ✅ UPDATED

**Test ID:** NEW-A1

**Priority:** 🔴 Critical

Run mode: First run with `expo start --clear`; subsequent runs without `--clear`.

**Steps:**

1. [ ] Fresh install, open app
2. [ ] See Splash screen (2s animation)
3. [ ] See Welcome screen (3 features)
4. [ ] Tap "Get Started" → Carousel
5. [ ] Swipe through 2 carouse3 slides
6. [ ] Tap "Next" → Auth screen
7. [ ] Tap "Continue with Google"
8. [ ] Complete OAuth → Survey Question 1/5
9. [ ] Answer all 5 questions:
   - Gender (required)
   - Interested In (required)
   - Outcome (required)
   - AI Style (required)
   - AI Language (required)
10. [ ] Tap "Next" on Q5 → Thanks screen
11. [ ] Tap "Let's Go!" → Openers screen

**Expected:**

- ✅ Each screen renders properly
- ✅ Navigation smooth (< 300ms transitions)
- ✅ No crashes or errors
- ✅ Profile saved with onboardingCompleted = true
- ✅ Total time: ~90-120 seconds

**Logs to Check:**

"Not signed in → Welcome screen"
"Auth success → Survey/1"
"Survey complete → Thanks"
"Onboarding complete → Main app"

---

### 2. Returning User - Skip Onboarding ✅ SAME AS BEFORE

**Test ID:** RETURN-B1

**Priority:** 🔴 Critical

Run mode: Without `--clear`.

**Steps:**

1. [ ] User completed onboarding previously
2. [ ] Close app
3. [ ] Reopen app

**Expected:**

- ✅ Splash → Openers screen (skip welcome/carousel/survey)
- ✅ Time: < 3 seconds
- ✅ Profile loaded from API

**Log:** "Onboarding complete → Main app"

---

### 3. Mid-Survey Exit & Resume ✅ UPDATED

**Test ID:** MID-SURVEY-A2

**Priority:** 🔴 Critical

Run mode: Without `--clear`.

**Steps:**

1. [ ] Fresh install → Welcome → Carousel → Auth
2. [ ] Start survey, answer questions 1-3
3. [ ] Force close app (swipe up)
4. [ ] Reopen app

**Expected:**

- ✅ Resume at Survey Question 1 (not Welcome)
- ✅ Answers 1-3 preserved (check onboarding store)
- ✅ Can continue from Q4

**Log:** "Survey in progress → Resume survey (answersCount: 3)"

---

### 4. Authenticated But No Survey Progress ✅ NEW TEST

**Test ID:** AUTH-NO-SURVEY

**Priority:** 🔴 Critical

Run mode: Without `--clear`.

**Steps:**

1. [ ] Fresh install → Welcome → Carousel → Auth
2. [ ] Complete OAuth successfully
3. [ ] **Force close app BEFORE starting survey**
4. [ ] Reopen app

**Expected:**

- ✅ Go back to Welcome screen (restart flow)
- ✅ User is already authenticated
- ✅ When reaching carousel → Skip auth → Go to Survey

**Log:** "Signed in but no survey progress → Welcome screen"

---

### 5. Skip Carousel ✅ NEW TEST

**Test ID:** CAROUSEL-SKIP

**Priority:** 🟡 High

Run mode: Without `--clear` (use `--clear` only if testing cold-start artifacts).

**Steps:**

1. [ ] Fresh install → Welcome → Carousel
2. [ ] Tap "Skip" on first carousel slide

**Expected:**

- ✅ If NOT authenticated → Go to Auth
- ✅ If authenticated → Go to Survey

**Log:** "Carousel skipped → [Auth/Survey]"

---

### 6. Already Authenticated at Carousel ✅ NEW TEST

**Test ID:** AUTH-SKIP-AT-CAROUSEL

**Priority:** 🟡 High

Run mode: Without `--clear`.

**Steps:**

1. [ ] User authenticated (previous session)
2. [ ] Open app → Welcome → Carousel
3. [ ] Tap "Next" on last carousel slide

**Expected:**

- ✅ Skip Auth screen (already signed in)
- ✅ Go directly to Survey

**Log:** "Already authenticated → Skip auth, Survey/1"

---

### 7. Logout → Login ✅ SAME AS BEFORE

**Test ID:** LOGOUT-LOGIN-B3

**Priority:** 🔴 Critical

Run mode: Without `--clear`.

**Steps:**

1. [ ] User in main app
2. [ ] Settings → Logout
3. [ ] Confirm logout → See Welcome screen
4. [ ] Navigate: Welcome → Carousel → Auth
5. [ ] Sign in with same account

**Expected:**

- ✅ After login, skip Welcome/Carousel
- ✅ Go directly to Openers (onboarding already complete)

**Log:** "Onboarding complete → Main app"

---

## 📊 Updated Test Matrix

| Category       | Old Count | New Count | Changed Tests                 |
| -------------- | --------- | --------- | ----------------------------- |
| Core Flows     | 5         | 7         | +2 (Welcome, Carousel)        |
| Returning User | 5         | 5         | No change                     |
| Mid-Onboarding | 3         | 4         | +1 (Auth but no survey)       |
| Navigation     | N/A       | 3         | NEW (Carousel skip, back nav) |
| Error Handling | 3         | 3         | No change                     |
| **TOTAL**      | **35**    | **40**    | **+5 new tests**              |

---

## 🎯 Recommended Action Plan

### Step 1: Update Documentation (5 min)

1. [ ] Add "🔄 UPDATE" section at top of file
2. [ ] Mark old test cases with "(DEPRECATED)"
3. [ ] Add new test cases for Welcome/Carousel/Thanks
4. [ ] Update A1 test case with new steps

### Step 2: Archive Old Tests (2 min)

1. [ ] Move deprecated tests to `## DEPRECATED TESTS` section at bottom
2. [ ] Keep them for reference but mark clearly

### Step 3: Prioritize Testing (10 min)

1. [ ] Run 7 critical tests (NEW-A1 through LOGOUT-LOGIN-B3)
2. [ ] Document results
3. [ ] Fix any blockers

---

## 🔄 UPDATED TEST CASES (v1.0 - New Flow)

// New tests above; old tests below remain for reference.

📋 Kompletan QA Test Plan - Authentication & Navigation Flow
Evo svih mogućih use case-ova koje treba testirati za produkciju:

🎯 CORE AUTHENTICATION FLOWS
A. NOVI KORISNIK (First Time User)
A1. Fresh Install → Sign Up → Complete Onboarding
Steps:

1. Install app (fresh)
2. Open app
3. See auth screen
4. Click "Sign up with Phone" (ili drugi method)
5. Complete phone verification
6. See onboarding welcome screen
7. Complete all onboarding steps
8. Submit onboarding
9. See app (openers screen)

Expected:
✅ No splash flash
✅ Smooth transition auth → onboarding → app
✅ Profile saved to backend
✅ onboardingCompleted = true

A2. Fresh Install → Sign Up → Exit During Onboarding
Steps:

1. Install app
2. Sign up
3. Start onboarding (complete 2-3 steps)
4. Force close app (swipe up)
5. Reopen app

Expected:
✅ Should resume from last onboarding step
✅ Onboarding answers preserved in store
✅ No data loss

A3. Sign Up → Network Error During Onboarding Submit
Steps:

1. Complete onboarding
2. Turn off internet
3. Click "Let's go!"
4. See error message
5. Turn on internet
6. Click "Try Again"

Expected:
✅ Error message shown (network error)
✅ Retry button works
✅ Profile saves successfully after retry
✅ Navigate to app

B. POSTOJEĆI KORISNIK (Returning User)
B1. Close App → Reopen (Signed In) ✅ TESTED
Steps:

1. User is signed in
2. Close app normally (home button)
3. Reopen app

Expected:
✅ Session restored automatically
✅ Profile loaded from API
✅ Navigate directly to openers screen
✅ No auth screen flash
✅ Time: < 2 seconds

B2. Force Kill App → Reopen (Signed In)
Steps:

1. User is signed in
2. Force kill app (swipe up from multitasking)
3. Reopen app

Expected:
✅ Session restored from Clerk cache
✅ Profile loaded from API
✅ Navigate to openers screen
✅ No auth screen flash

B3. Logout → Login (Same Device) ⚠️ NEEDS TESTING
Steps:

1. User is signed in
2. Go to Settings → Account
3. Click "Logout"
4. Confirm logout
5. See auth screen
6. Click "Sign in"
7. Complete sign in

Expected:
✅ Logout clears all stores
✅ Navigate to auth screen
✅ After login, profile loads from API
✅ Navigate to openers screen
✅ No cached data from previous session

B4. Reopen App After 24 Hours (Session Expired)
Steps:

1. User is signed in
2. Close app
3. Wait 24+ hours (or manually expire Clerk session)
4. Reopen app

Expected:
✅ Session expired detected
✅ Navigate to auth screen
✅ User prompted to sign in again
✅ No crash or error

B5. Reopen App After 7 Days (Long Absence)
Steps:

1. User is signed in
2. Close app
3. Wait 7+ days
4. Reopen app

Expected:
✅ Session may or may not be valid (depends on Clerk config)
✅ If valid: Navigate to app
✅ If expired: Navigate to auth screen
✅ No crash

C. ONBOARDING EDGE CASES
C1. Complete Onboarding → Backend Returns 400 Error
Steps:

1. Complete onboarding
2. Submit with invalid data (mock backend error)
3. See validation error

Expected:
✅ Error message shown
✅ Retry button available
✅ No infinite loop
✅ No navigation to app until successful

C2. Complete Onboarding → API Timeout
Steps:

1. Complete onboarding
2. Slow down network (Network Link Conditioner)
3. Submit onboarding
4. Wait for timeout (10s)

Expected:
✅ Timeout warning shown
✅ Retry option available
✅ No crash

C3. User Deleted From Backend → Reopen App
Steps:

1. User is signed in
2. Admin deletes user from backend database
3. User closes app
4. User reopens app

Expected:
✅ API returns 404
✅ Profile reset to initial state
✅ User redirected to onboarding
✅ No crash

D. MULTIPLE DEVICES
D1. Sign In on Device A → Sign In on Device B
Steps:

1. Sign in on iPhone
2. Complete onboarding
3. Close app
4. Sign in on iPad with same account
5. Open app on iPhone again

Expected:
✅ Both devices see same profile
✅ onboardingCompleted synced
✅ No conflicts

D2. Change Profile on Device A → Open Device B
Steps:

1. Signed in on both devices
2. On Device A: Change profile setting
3. On Device B: Reopen app

Expected:
✅ Device B loads fresh profile from API
✅ Changes reflected

E. NETWORK CONDITIONS
E1. Cold Start with No Internet
Steps:

1. Turn off WiFi and cellular
2. Open app (signed in)

Expected:
✅ App opens but shows "No connection" message
✅ Cached profile shown (if available)
✅ Retry option available
✅ No crash

E2. Sign In with Weak Network
Steps:

1. Enable slow 3G simulation
2. Attempt sign in
3. Wait for response

Expected:
✅ Loading indicator shown
✅ Timeout after 10s
✅ Error message shown
✅ Retry option available

E3. Internet Disconnects Mid-Session
Steps:

1. User is signed in and using app
2. Turn off internet
3. Navigate between screens

Expected:
✅ Offline queue captures failed requests
✅ Toast shows "You're offline"
✅ App doesn't crash
✅ Requests retry when online

F. RACE CONDITIONS & TIMING
F1. Rapid App Open/Close Cycles
Steps:

1. Open app
2. Immediately close (before splash completes)
3. Reopen
4. Repeat 5 times

Expected:
✅ No crash
✅ No duplicate API calls
✅ Eventually shows correct screen

F2. Sign In → Immediately Force Close
Steps:

1. Click "Sign in"
2. Immediately force close app mid-authentication
3. Reopen app

Expected:
✅ Auth state recovered or back to auth screen
✅ No partial state
✅ No crash

F3. Multiple Rapid Logouts
Steps:

1. Signed in
2. Click logout
3. Immediately click logout again (spam)
4. Click 5 times rapidly

Expected:
✅ Only one logout processed
✅ No multiple redirects
✅ Clean auth screen shown

G. DEEP LINKING
G1. Deep Link to Protected Route (Not Signed In)
Steps:

1. User is signed out
2. Tap deep link: charisma://app/sessions
3. App opens

Expected:
✅ Redirect to auth screen
✅ After sign in, show sessions screen
✅ No crash

G2. Deep Link to Onboarding (Already Completed)
Steps:

1. User completed onboarding
2. Tap deep link: charisma://onboarding/welcome
3. App opens

Expected:
✅ Redirect to app (openers)
✅ Don't show onboarding again

G3. Deep Link to Non-Existent Route
Steps:

1. Tap deep link: charisma://app/fake-screen
2. App opens

Expected:
✅ Fallback to default screen (openers)
✅ Or show "Page not found"
✅ No crash

H. STORAGE & CACHE
H1. Clear App Cache (iOS Settings)
Steps:

1. Signed in
2. Go to iOS Settings → Charisma → Clear Cache
3. Reopen app

Expected:
✅ Session restored from Clerk
✅ Profile reloaded from API
✅ Navigate to app

H2. Delete & Reinstall App (Same Account)
Steps:

1. Signed in
2. Delete app
3. Reinstall from App Store
4. Open app
5. Sign in with same account

Expected:
✅ Profile loaded from backend
✅ onboardingCompleted preserved
✅ No need to redo onboarding

H3. Storage Full Error
Steps:

1. Fill device storage (< 100MB free)
2. Open app
3. Complete onboarding

Expected:
✅ Error handling for storage write failure
✅ Graceful degradation
✅ User notified

I. PERMISSION & OS EVENTS
I1. Notification Permission Denied
Steps:

1. Complete onboarding
2. Click "Let's go!"
3. Notification prompt appears
4. Click "Don't Allow"

Expected:
✅ App continues to openers screen
✅ No crash or error
✅ Notification permission can be requested later

I2. App Backgrounded During API Call
Steps:

1. Click "Let's go!" to submit onboarding
2. Immediately press home button (background app)
3. Wait 5 seconds
4. Reopen app

Expected:
✅ API call completes in background
✅ Navigate to app when reopened
✅ Or show loading state if still processing

I3. Phone Call Interrupts Onboarding
Steps:

1. On onboarding screen
2. Receive phone call
3. Answer call
4. End call, return to app

Expected:
✅ Onboarding state preserved
✅ No data loss
✅ Continue where left off

J. PAYWALL FLOW
J1. Complete Onboarding → Paywall Shows
Steps:

1. Complete onboarding
2. Click "Let's go!"
3. Wait for navigation

Expected:
✅ Navigate to openers screen FIRST
✅ Then paywall modal appears (500ms delay)
✅ Can dismiss paywall
✅ Can use app after dismissing

J2. Subscribe on Paywall → Close App → Reopen
Steps:

1. Paywall shows
2. Click "Start free trial"
3. Complete purchase
4. Close app
5. Reopen app

Expected:
✅ Subscription active
✅ Paywall doesn't show again
✅ Premium features unlocked

K. ERROR RECOVERY
K1. API Returns 500 Error
Steps:

1. Mock backend 500 error
2. Complete onboarding
3. Click "Let's go!"

Expected:
✅ Retry logic (exponential backoff)
✅ After 3 retries, show error
✅ Retry button available

K2. Clerk Service Down
Steps:

1. Mock Clerk API failure
2. Try to sign in

Expected:
✅ Error message shown
✅ Retry option available
✅ No crash

K3. Corrupted Local Storage
Steps:

1. Manually corrupt AsyncStorage data
2. Open app

Expected:
✅ App detects corrupted data
✅ Resets to clean state
✅ User can sign in again

📊 Test Matrix Summary
Category

# Tests

Priority
Core Auth
5
🔴 Critical
Returning User
5
🔴 Critical
Onboarding
3
🟡 High
Multiple Devices
2
🟡 High
Network
3
🟡 High
Race Conditions
3
🟡 High
Deep Linking
3
🟢 Medium
Storage
3
🟢 Medium
Permissions
3
🟢 Medium
Paywall
2
🟢 Medium
Error Recovery
3
🟡 High
TOTAL
35

-

🚀 Testing Priority Order
Phase 1: Must Pass (Ship Blockers) 🔴
✅ B1. Close → Reopen
⚠️ B3. Logout → Login
A1. Fresh install → Complete onboarding
B4. Session expired
E1. No internet on start

Phase 2: Should Pass (High Priority) 🟡
A2. Exit during onboarding
A3. Network error during submit
C3. User deleted from backend
D1. Multiple devices
F1. Rapid open/close
K1. API 500 error

Phase 3: Nice to Pass (Medium Priority) 🟢
All remaining tests

🧪 Kako Testirati?
Manual Testing Checklist:

- [ ] Run on physical iPhone (not simulator)
- [ ] Run on physical Android (not emulator)
- [ ] Test with slow 3G network
- [ ] Test with airplane mode
- [ ] Test with background app refresh off
- [ ] Test with low battery mode
- [ ] Force kill app between tests
- [ ] Clear cache between critical tests

### 📦 Kada koristiti `expo start --clear` (a kada NE)

Dodaj ovo pravilo u svoj test setup, jer utiče na to koje bagove možeš da reprodukuješ.

KORISTI `--clear` (namerno cold start, briše Metro/asset cache):

- Kada testiraš cold‑start artifakte: splash bljesak, asset flicker, font FOIT/FOIT.
- Posle promene verzija RN/Expo/native modula, ili promene fontova/slika.
- Kada sumnjaš na korumpiran cache/bundy (čudne greške koje nestanu posle reinstall).
- Prvi run „Fresh Install“ scenarija (NEW-A1) – opcionalno, da uhvatiš cold‑start edge case‑ove.

NE KORISTI `--clear` (ostavi warm cache za realističniji UX):

- Većina regresionih testova navigacije i vraćanja u app: RETURN-B1, LOGOUT-LOGIN-B3.
- Mid‑survey resume i persistence: MID-SURVEY-A2.
- Authenticated but no survey progress: AUTH-NO-SURVEY.
- Performance merenja tranzicija (NEW-A1 koraci 4→14) posle prvog učitavanja.
- Device background/foreground, deep linking, offline/online retry scenariji.

Praktično:

- Za „Fresh Install“ runde: `expo start --clear` (jednom), zatim više run‑ova bez `--clear`.
- Za dnevni dev ciklus: samo `expo start` (bez `--clear`). `--clear` koristiš ciljano kad treba cold‑start.

Automated Testing (Future):
// e2e/**tests**/auth-flow.test.ts
describe('Authentication Flow', () => {
it('B1: should restore session on reopen', async () => {
// Test implementation
});

it('B3: should handle logout → login', async () => {
// Test implementation
});
});

📝 Bug Report Template
Kada nađeš bug, koristi ovaj format:
**Test Case:** B3 - Logout → Login
**Device:** iPhone 15 Pro, iOS 17.2
**Build:** 1.0.0+1

**Steps to Reproduce:**

1. User is signed in
2. Logout from Settings
3. Sign in again

**Expected:** Navigate to openers screen
**Actual:** Stuck on auth screen

**Logs:** [paste relevant logs]
**Screenshots:** [attach if applicable]

**Priority:** 🔴 Critical

✅ Bottom Line
35 test cases pokrivaju sve kritične scenarije za production.
Prioritiziraj:
🔴 5 critical tests - MORA da prođe
🟡 15 high priority - TREBALO BI da prođe
🟢 15 medium priority - NICE TO HAVE
Trenutni Status:
✅ B1 tested & passing
⚠️ B3 needs confirmation (possible issue sa navigation)
❓ Ostali nisu testirani
Javi mi koje test case želiš da prioritizujem za sledeće debug sessione! 🚀
