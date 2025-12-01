# Authentication Flow Documentation

> **Current Flow (v1.1, implemented)**  
> `Splash → Welcome → Carousel → Auth → Survey → Thanks → App`
>
> **High-level routing rule (root guard):**
>
> ```typescript
> // RootLayoutNav redirect target (simplified)
> if (isLoaded && !isSignedIn) return "/(onboarding)/welcome"; // unauthenticated
> if (isLoaded && isSignedIn && !isProfileFreshFromApi) return null; // wait for API
> if (!profile?.id) return "/(onboarding)/welcome"; // no profile → onboarding
> if (!profile.onboardingCompleted) return "/(onboarding)/welcome";
> return "/(app)/(tabs)/openers"; // fully onboarded
> ```
>
> **Core test cases to maintain:**
>
> - Welcome + carousel render and navigation
> - Auth skip when already authenticated at carousel or resume
> - Survey resume after app close (partial answers)
> - Thanks → App navigation after survey completion
> - RootLayout/AppRoot routing for all auth/profile states

## Table of Contents

1. [Overview](#overview)
2. [Key Concepts](#key-concepts)
   - [isProfileFreshFromApi](#1-isprofilefreshfromapi)
   - [Triple Guard System](#2-triple-guard-system)
     - [Layer 1: RootLayoutNav](#layer-1-rootlayoutnav-clientapp_layouttsx)
     - [Layer 2: AppRoot](#layer-2-approot-clientappappindextsx)
     - [Layer 3: Individual Screen Guards](#layer-3-individual-screen-guards)
3. [Edge Cases](#edge-cases)
   - [API Timeout (10 seconds)](#1-api-timeout-10-seconds)
   - [Profile Not Found (404)](#2-profile-not-found-404)
   - [Cached Stale Data](#3-cached-stale-data)
   - [Logout Multi-Account Scenario](#4-logout-multi-account-scenario)
4. [Backend Authentication Middleware](#backend-authentication-middleware)
5. [Data Flow](#data-flow)
6. [API Client Configuration](#api-client-configuration)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Key Takeaways](#key-takeaways)

---

## Overview

The authentication system in Charisma uses a **three-layer guard system** to ensure users are routed correctly based on their authentication and profile state. The system prioritizes backend data over cached data to prevent stale state issues.

## New User Flow (v1.1 - Current) {#new-user-flow}

### Flow Diagram

```
┌─────────────┐
│   Splash    │ (animated, fonts loading, RootLayoutNav boot)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Welcome   │ (3 features: Openers, Screenshots, Expert advice)
└──────┬──────┘  Button: "Get Started" → Carousel
       │
       ▼
┌─────────────┐
│  Carousel   │ (2 slides: Value proposition + Social proof)
└──────┬──────┘  "Next" or "Skip"
       │
       ▼
┌─────────────┐
│    Auth     │ (Google/Apple OAuth, only if NOT already signed in)
└──────┬──────┘  Auto-skipped if user isSignedIn
       │
       ▼
┌─────────────┐
│   Survey    │ (5 questions: Gender, Interested In, Outcome, AI Style, Language)
└──────┬──────┘  Progress bar + Next/Back
       │
       ▼
┌─────────────┐
│   Thanks    │ ("You're all set! 💬")
└──────┬──────┘  Button: "Let's Go!"
       │
       ▼
┌─────────────┐
│     App     │ (Main app - Openers tab)
└─────────────┘
```

### New Screens

#### 1. Welcome Screen (`app/(onboarding)/welcome.tsx`)

**Purpose:** Show the value proposition _and_ short-circuit onboarding when possible.

**Features:**

- Logo and branding
- 3 feature highlights:
  1. AI-powered openers and replies
  2. Upload chat screenshots for feedback
  3. Advice from 1,000+ expert articles
- "Get Started" button → Carousel

**Routing / Guards (current code):**

```typescript
// If onboarding already complete, skip welcome entirely
useEffect(() => {
  if (
    isLoaded &&
    isSignedIn &&
    isProfileFreshFromApi &&
    profile?.onboardingCompleted
  ) {
    router.replace("/(app)/(tabs)/openers");
  }
}, [isLoaded, isSignedIn, isProfileFreshFromApi, profile?.onboardingCompleted]);

// If user started onboarding earlier, skip welcome + carousel, resume at Q1
useEffect(() => {
  if (!isLoaded || !isSignedIn || !_hasHydrated) return;
  const hasPartialAnswers = answers && Object.keys(answers).length > 0;
  if (!profile?.onboardingCompleted && hasPartialAnswers) {
    router.replace("/(onboarding)/survey/1");
  }
}, [isLoaded, isSignedIn, _hasHydrated, answers, profile?.onboardingCompleted]);

// Primary CTA
const handleGetStarted = () => {
  router.replace("/(onboarding)/carousel");
};
```

---

#### 2. Carousel Screen (`app/(onboarding)/carousel.tsx`)

**Purpose:** Showcase product value and build trust before auth.

**Features:**

- 2 swipeable slides:
  1. Value Proposition: What the app does
  2. Social Proof: trust signal
- "Next" button (last slide) or "Skip" button
- Progress indicators

**Routing / Guards (current code):**

```typescript
// If user already started onboarding earlier, skip carousel entirely
useEffect(() => {
  if (!isSignedIn || !_hasHydrated) return;
  const hasPartialAnswers = answers && Object.keys(answers).length > 0;
  if (hasPartialAnswers) {
    router.replace("/(onboarding)/survey/1");
  }
}, [isSignedIn, _hasHydrated, answers]);

const handleNext = () => {
  const nextIndex = currentIndex + 1;
  if (nextIndex < CAROUSEL_CONFIG.length) {
    scrollViewRef.current?.scrollTo({
      x: nextIndex * slideWidth,
      animated: true,
    });
  } else {
    if (isSignedIn) {
      router.replace("/(onboarding)/survey/1"); // skip auth when already signed in
    } else {
      router.replace("/(auth)"); // go to auth
    }
  }
};

const handleSkip = () => {
  if (isSignedIn) {
    router.replace("/(onboarding)/survey/1");
  } else {
    router.replace("/(auth)");
  }
};
```

---

#### 3. Survey Screen (`app/(onboarding)/survey/[step].tsx`)

**Purpose:** Collect 5 essential profile fields (formerly "onboarding").

**Questions:**

1. Gender (required)
2. Interested In (required)
3. Outcome - What are you looking for? (required)
4. AI Style - How should Charisma respond? (required)
5. AI Language (required)

Deferred to settings:

- Age
- Motivation (multi-select)
- Challenge (multi-select)

**Navigation:**

- Questions 1–4: → Next question
- Question 5: → Thanks screen (`/(onboarding)/complete`) via onboarding layout

**Auth Guard (current code):**

```typescript
// app/(onboarding)/survey/[step].tsx
const { isSignedIn, isLoaded } = useClerkToken();

if (!isLoaded) {
  return null; // Wait for Clerk to load
}
if (!isSignedIn) {
  return <Redirect href="/(auth)" />;
}
```

---

#### 4. Thanks Screen (`app/(onboarding)/complete.tsx`)

Purpose: Celebrate completion and transition to app

Features:

- Success message: "You're all set! 💬"
- "Let's Go!" button → Main app

Backend Actions:

- Sets `onboardingCompleted = true` in database
- User profile is now complete

---

### Updated Routing Logic

Before (Deprecated):

```typescript
if (isLoaded && !isSignedIn) return "/(auth)"; // ❌ Auth barrier first
```

After (Current):

```typescript
if (isLoaded && !isSignedIn) return "/(onboarding)/welcome"; // ✅ Show value first
```

Full Logic:

```typescript
const redirectTarget = useMemo(() => {
  if (!isInitialRouteDetermined) return null;
  if (isAlreadyInApp || isAlreadyInOnboarding) return null;

  // NEW: Show welcome to unauthenticated users
  if (isLoaded && !isSignedIn) return "/(onboarding)/welcome";

  // Wait for fresh profile from API
  if (isLoaded && isSignedIn && !isProfileFreshFromApi) return null;

  // Check profile completeness
  if (!profile?.id) return "/(onboarding)/welcome";
  if (!profile.onboardingCompleted) return "/(onboarding)/welcome";

  // User fully onboarded → Main app
  return "/(app)/(tabs)/openers";
}, [
  isInitialRouteDetermined,
  isAlreadyInApp,
  isAlreadyInOnboarding,
  isLoaded,
  isSignedIn,
  isProfileFreshFromApi,
  profile?.id,
  profile?.onboardingCompleted,
]);
```

---

### New User Scenarios

Scenario 1: Fresh Install (Happy Path)

```
1. App opens → Splash (2s)
2. RootLayoutNav: isSignedIn = false → '/(onboarding)/welcome'
3. User sees Welcome → taps "Get Started"
4. Navigate to Carousel
5. User swipes through 2 slides → taps "Next"
6. Carousel checks: isSignedIn = false → Navigate to Auth
7. User taps "Continue with Google" → Auth success
8. Auth screen: useEffect detects isSignedIn = true → Navigate to Survey/1
9. User answers 5 questions
10. Submit → Backend sets onboardingCompleted = true
11. Navigate to Thanks screen
12. User taps "Let's Go!" → Navigate to Openers
```

Scenario 2: Mid-Onboarding App Close

```
1. User completes: Welcome → Carousel → Auth
2. User answers 2/5 survey questions
3. User force closes app (swipe up)
4. User reopens app
5. RootLayoutNav: isSignedIn = true, isProfileFreshFromApi loads...
6. Profile loaded: onboardingCompleted = false
7. Onboarding store has answers (2/5)
8. Redirect to Survey/1 (resume from last question)
```

Scenario 3: Authenticated But No Survey Progress

```
1. User completes: Welcome → Carousel → Auth
2. User force closes app BEFORE starting survey
3. User reopens app
4. RootLayoutNav: isSignedIn = true, onboardingCompleted = false
5. Onboarding store is empty (no answers)
6. Redirect to '/(onboarding)/welcome' (restart flow)
7. User navigates: Welcome → Carousel
8. Carousel checks: isSignedIn = true → Skip Auth → Go to Survey
```

Scenario 4: Carousel Skip

```
1. User on Carousel slide 1
2. User taps "Skip"
3. Carousel checks: isSignedIn?
   - If false: → Auth
   - If true: → Survey/1
```

---

### Edge Cases

Auth Already Complete at Carousel

```typescript
// app/(onboarding)/carousel.tsx

const handleNext = () => {
  if (isSignedIn) {
    router.replace("/(onboarding)/survey/1"); // Skip auth
  } else {
    router.replace("/(auth)");
  }
};
```

Prevents: Showing auth screen to already-authenticated users

Survey Auth Guard

```typescript
// app/(onboarding)/survey/[step].tsx

if (isLoaded && !isSignedIn) {
  return <Redirect href="/(auth)" />;
}
```

---

---

## Key Concepts

### 1. `isProfileFreshFromApi` {#1-isprofilefreshfromapi}

**Purpose:** Ensures routing decisions are based on server state, not cached data.

**Location:** `client/src/store/userProfile.store.ts`

```typescript
interface UserProfileState {
  profile: UserProfile;
  isHydrated: boolean; // Store is loaded from persistent storage
  isProfileFreshFromApi: boolean; // Profile is fresh from API
}
```

**Lifecycle:**

- `false` on app initialization (rehydrated data is stale)
- `false` when Zustand rehydrates from storage
- `true` only after successful API call to `/users/me`
- `false` on sign-out

**Why it matters:**
Without this flag, the app would use stale cached data for routing decisions, leading to users being shown the wrong screen.

---

### 2. Triple Guard System {#2-triple-guard-system}

#### Layer 1: RootLayoutNav (`client/app/_layout.tsx`) {#layer-1-rootlayoutnav-clientapp_layouttsx}

Primary authentication guard that handles initial app startup and authentication state.

**Guards:**

1. Wait for auth and hydration systems
2. Redirect to `/(auth)` if not signed in
3. Wait for `isProfileFreshFromApi` to ensure fresh data
4. Redirect to onboarding if profile doesn't exist
5. Redirect to onboarding if not completed
6. Redirect to `/(app)` if authenticated and onboarded

```105:146:client/app/_layout.tsx
  useEffect(() => {
    // Guard 1: Wait for the main systems to be ready
    if (!isAuthInitialized || !isHydrated) {
      logger.debug('Waiting for auth/hydration...', {
        isAuthInitialized,
        isHydrated,
      });
      return;
    }

    // Guard 2: If not signed in, route is clear
    if (!isSignedIn) {
      logger.debug('Not signed in, route determined.');
      setInitialRouteDetermined(true);
      return;
    }

    // Guard 3: Only skip loading if profile is already fresh from API
    if (profile?.id && isProfileFreshFromApi) {
      logger.debug('Profile already exists in store, route determined.');
      if (!profileLoadAttemptedRef.current) {
        profileLoadAttemptedRef.current = true;
      }
      setInitialRouteDetermined(true);
      return;
    }

    // Guard 4: If already loaded or loading, don't try again
    if (profileLoadAttemptedRef.current || isLoadingProfileRef.current) {
      logger.debug('Profile already loading/loaded');
      return;
    }

    // Main logic: Load profile from API
    logger.debug('Loading profile from API...');
    isLoadingProfileRef.current = true;
    profileLoadAttemptedRef.current = true;

    let timeoutId: NodeJS.Timeout;

    const loadProfile = async () => {
      try {
        // Timeout protection
        timeoutId = setTimeout(() => {
          if (mountedRef.current) {
            logger.warn('Profile load timeout - continuing anyway');
            setInitialRouteDetermined(true);
          }
        }, PROFILE_LOAD_TIMEOUT);
```

**Profile Loading Logic:**

```143:192:client/app/_layout.tsx
    const loadProfile = async () => {
      try {
        // Timeout protection
        timeoutId = setTimeout(() => {
          if (mountedRef.current) {
            logger.warn('Profile load timeout - continuing anyway');
            setInitialRouteDetermined(true);
          }
        }, PROFILE_LOAD_TIMEOUT);

        const token = await getAuthToken();

        if (!mountedRef.current) {
          return;
        }

        if (token) {
          await loadUserProfileFromApi(token);
        }
      } catch (error: unknown) {
        if (!mountedRef.current) {
          return;
        }

        // 404 is expected if profile doesn't exist yet
        const is404 =
          error &&
          typeof error === 'object' &&
          'response' in error &&
          error.response &&
          typeof error.response === 'object' &&
          'status' in error.response &&
          error.response.status === 404;

        if (!is404) {
          logger.warn('Error loading profile', {
            error: error instanceof Error ? error.message : String(error),
          });
        }
      } finally {
        if (timeoutId) {
          clearTimeout(timeoutId);
        }
        isLoadingProfileRef.current = false;

        if (mountedRef.current) {
          setInitialRouteDetermined(true);
        }
      }
    };

    loadProfile();
  }, [
    isSignedIn,
    isHydrated,
    isAuthInitialized,
    profile?.id,
    getAuthToken,
    loadUserProfileFromApi,
  ]);
```

**Final Routing:**

```261:285:client/app/_layout.tsx
  // Routing guards
  if (!isSignedIn) {
    return <Redirect href="/(auth)" />;
  }

  // Guard: Waiting for fresh profile data from API
  // This prevents routing based on stale cached data
  if (!isProfileFreshFromApi) {
    logger.debug('Waiting for fresh profile from API...', {
      hasProfileId: !!profile?.id,
      onboardingCompleted: profile?.onboardingCompleted,
    });
    return null; // Keep showing splash
  }

  if (!profile?.id) {
    return <Redirect href="/(onboarding)/welcome" />;
  }

  // Guard: Onboarding not completed
  if (!profile.onboardingCompleted) {
    return <Redirect href="/(onboarding)/welcome" />;
  }

  return <Redirect href="/(app)" />;
```

#### Layer 2: AppRoot (`client/app/(app)/index.tsx`) {#layer-2-approot-clientappappindextsx}

Secondary guard for the authenticated app section.

```6:33:client/app/(app)/index.tsx
export default function AppRoot() {
  const { isSignedIn, isInitialized } = useClerkToken();
  const { profile, isHydrated, isProfileFreshFromApi } = useUserProfileStore();

  //  Wait for auth and store to be ready
  if (!isLoaded || !isInitialized || !isHydrated) {
    return null; // Still loading
  }

  //  This should never happen (RootLayoutNav guards this)
  // But as safety fallback, redirect to auth
  if (!isSignedIn) {
    return <Redirect href="/(auth)" />;
  }

  //  Wait for profile to be loaded from API
  if (!isProfileFreshFromApi) {
    return null; // Still loading profile
  }

  if (!profile?.id || !profile?.onboardingCompleted) {
    logger.warn('[AppRoot] Missing profile or onboarding, redirecting');
    return <Redirect href="/(onboarding)/welcome" />;
  }

  //  Finally, redirect to tabs
  return <Redirect href="/(app)/(tabs)" />;
}
```

#### Layer 3: Individual Screen Guards {#layer-3-individual-screen-guards}

Each protected screen (e.g., in `(app)` group) can implement additional guards if needed.

---

## Edge Cases {#edge-cases}

### 1. API Timeout (10 seconds) {#1-api-timeout-10-seconds}

**Scenario:** API doesn't respond within 10 seconds

**Handling:**

```64:64:client/app/_layout.tsx
const PROFILE_LOAD_TIMEOUT = 10000;
```

```146:151:client/app/_layout.tsx
        timeoutId = setTimeout(() => {
          if (mountedRef.current) {
            logger.warn('Profile load timeout - continuing anyway');
            setInitialRouteDetermined(true);
          }
        }, PROFILE_LOAD_TIMEOUT);
```

**Behavior:** App continues after timeout and uses cached data if available. This prevents users from being stuck on the splash screen due to network issues.

---

### 2. Profile Not Found (404) {#2-profile-not-found-404}

**Scenario:** Backend returns 404 when user profile doesn't exist

**Backend Behavior:**

```68:74:backend/src/modules/auth/loadUser.middleware.ts
    // User must exist at this point
    if (!user) {
      throw new AppError(
        'User profile not found in database. Please complete signup process.',
        404,
        ErrorCodes.RESOURCE_NOT_FOUND
      );
    }
```

**Client Handling:**

```279:303:client/src/store/userProfile.store.ts
        } catch (error: unknown) {
          // Handle 404 - user profile doesn't exist in database
          // This happens when user was deleted or database is out of sync
          if (
            error &&
            typeof error === 'object' &&
            'response' in error &&
            error.response &&
            typeof error.response === 'object' &&
            'status' in error.response &&
            error.response.status === 404
          ) {
            logger.warn(
              'User profile not found (404), resetting to initial state'
            );
            // Reset profile to initial state and mark as fresh (we confirmed 404)
            set({ profile: initialState, isProfileFreshFromApi: true });
            return false;
          }

          // For other errors, log but don't reset
          logger.error('Failed to load user profile from API', {
            error: error instanceof Error ? error.message : String(error),
          });
          return false;
        }
```

**Important:** 404 is treated as a **valid state**, not an error. The app resets the profile to initial state and marks it as "fresh" so routing can proceed to onboarding.

---

### 3. Cached Stale Data {#3-cached-stale-data}

**Problem:** Zustand persists profile data, which could be outdated when user returns to the app.

**Solution:** `isProfileFreshFromApi` flag ensures we always wait for fresh API data before routing.

**Flow:**

1. App starts → Zustand rehydrates → `isProfileFreshFromApi: false`
2. User appears authenticated (has cached profile with `onboardingCompleted: true`)
3. RootLayoutNav sees `isProfileFreshFromApi: false` → waits
4. API call completes → updates profile → sets `isProfileFreshFromApi: true`
5. Now routing happens with fresh data

**If API fails:** The user is shown cached data after timeout, but this is acceptable for offline scenarios.

---

### 4. Logout Multi-Account Scenario {#4-logout-multi-account-scenario}

**Scenario:** User logs out and logs in with a different account

**Handling:**

```413:427:client/app/_layout.tsx
function AuthAwareEffects() {
  const { isSignedIn } = useClerkToken();
  const resetProfile = useUserProfileStore(s => s.resetProfile);
  const resetOnboarding = useOnboardingStore(s => s.resetOnboarding);

  useEffect(() => {
    if (!isSignedIn) {
      resetProfile();
      // Reset onboarding store
      resetOnboarding();
    }
  }, [isSignedIn, resetProfile]);

  return null;
}
```

**Reset Implementation:**

```306:311:client/src/store/userProfile.store.ts
      resetProfile: () => {
        set({ profile: initialState, isProfileFreshFromApi: false });
        // Clear persisted storage immediately
        zustandSecureStorage.removeItem('user-profile-storage-v2');
        logger.info('Profile reset and cleared from storage');
      },
```

**Flow:**

1. User signs out
2. `AuthAwareEffects` detects `isSignedIn: false`
3. Profile and onboarding stores are reset
4. Storage is cleared
5. User signs in with different account
6. Fresh profile is loaded from API for the new account

---

## Backend Authentication Middleware {#backend-authentication-middleware}

### `loadUser` Middleware

**Location:** `backend/src/modules/auth/loadUser.middleware.ts`

**Purpose:** Ensures user exists in database before allowing API access. Automatically restores soft-deleted users.

**Flow:**

```14:105:backend/src/modules/auth/loadUser.middleware.ts
export const loadUser = async (
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> => {
  try {
    if (!req.auth?.userId) {
      throw new AppError(
        'User not authenticated',
        401,
        ErrorCodes.AUTH_INVALID_TOKEN
      );
    }

    // Try to find active user
    let user = await userService.findUserByClerkId(req.auth.userId);

    // If user not found, check if they're soft-deleted
    if (!user) {
      const softDeletedUser =
        await userService.findUserByClerkIdIncludingDeleted(req.auth.userId);

      if (softDeletedUser) {
        // Automatically restore soft-deleted user
        const restoreResult = await userService.restoreUser(
          req.auth.userId,
          false
        );

        if (typeof restoreResult === 'boolean') {
          // This should never happen, but handle gracefully
          logger.error({
            message: 'Unexpected restoreUser result type',
            clerkId: req.auth.userId,
            resultType: typeof restoreResult,
          });
          throw new AppError(
            'Failed to restore user account',
            500,
            ErrorCodes.INTERNAL_SERVER_ERROR
          );
        }

        user = restoreResult;

        logger.info({
          message: 'Soft-deleted user restored automatically',
          userId: user.id,
          clerkId: req.auth.userId,
        });
      }
    }

    // User must exist at this point
    if (!user) {
      throw new AppError(
        'User profile not found in database. Please complete signup process.',
        404,
        ErrorCodes.RESOURCE_NOT_FOUND
      );
    }

    // Attach user to request
    req.user = user;

    logger.debug({
      message: 'User loaded successfully',
      userId: user.id,
      clerkId: req.auth.userId,
    });

    next();
  } catch (error) {
    logger.error({
      message: 'Error loading user',
      error: error instanceof Error ? error.message : 'Unknown error',
      userId: req.auth?.userId,
    });

    if (error instanceof AppError) {
      next(error);
    } else {
      next(
        new AppError(
          'Failed to load user profile',
          500,
          ErrorCodes.INTERNAL_SERVER_ERROR
        )
      );
    }
  }
};
```

**Key Features:**

1. Validates authentication
2. Searches for active user
3. **Auto-restores soft-deleted users** (graceful account recovery)
4. Returns 404 if user doesn't exist
5. Attaches user to request for downstream handlers

---

## Data Flow {#data-flow}

### App Startup (Authenticated User)

```
┌─────────────────────────────────────────────────────────────────┐
│                    1. APP STARTS                                │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. RootLayoutNav detects:                                      │
│     - isAuthInitialized: false                                  │
│     - isHydrated: false                                         │
│     → Shows splash screen                                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. Zustand rehydrates from storage                             │
│     - isHydrated: true                                          │
│     - isProfileFreshFromApi: false (CRITICAL!)                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. RootLayoutNav detects:                                      │
│     - isSignedIn: true                                          │
│     - profile.id exists (cached)                                │
│     - isProfileFreshFromApi: false                              │
│     → Loads profile from API                                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. API Call: GET /users/me                                     │
│     │                                                           │
│     ├─ Success → Updates profile                                │
│     │  sets isProfileFreshFromApi: true                         │
│     │                                                           │
│     └─ 404 → Resets profile to initial state                   │
│        sets isProfileFreshFromApi: true                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. RootLayoutNav checks routing:                              │
│     - isProfileFreshFromApi: true ✓                            │
│     - profile exists?                                           │
│     - onboardingCompleted?                                      │
│     → Redirects accordingly                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## API Client Configuration {#api-client-configuration}

**Timeout:** 10 seconds

```100:105:client/src/lib/api/apiClient.ts
const apiClient: AxiosInstance = axios.create({
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});
```

**Error Handling for 404:**

```102:131:client/src/lib/api/userApi.ts
export const getUserProfile = async (
  token: string
): Promise<UserApiResponse> => {
  try {
    const authApi = createAuthenticatedRequest(token);
    const response = await authApi.get<UserApiResponse>('/users/me');
    return response.data;
  } catch (error: unknown) {
    // Handle 404 as expected scenario (new user without profile)
    // Don't log it as an error - Zustand store will handle it
    if (
      error &&
      typeof error === 'object' &&
      'response' in error &&
      error.response &&
      typeof error.response === 'object' &&
      'status' in error.response &&
      error.response.status === 404
    ) {
      // Silently pass 404 to the caller without logging it as ERROR
      throw error;
    }

    // For all other errors, log them
    logger.error('getUserProfile failed', {
      error: error instanceof Error ? error.message : String(error),
    });
    throw error; // Re-throw the original Axios error to preserve status codes, response body, etc.
  }
};
```

---

## Best Practices {#best-practices}

1. **Always check `isProfileFreshFromApi` before routing** - cached data is not trusted
2. **Handle 404 gracefully** - it's a valid state, not an error
3. **Use timeouts for API calls** - prevent indefinite loading
4. **Reset state on logout** - clear all user data
5. **Backend is source of truth** - never rely only on cached data for auth decisions

---

## Troubleshooting {#troubleshooting}

### User stuck on splash screen

- Check if API call is timing out
- Verify network connectivity
- Check logs for error messages

### User routed to wrong screen

- Verify `isProfileFreshFromApi` is `true` before routing
- Check if profile data is stale
- Ensure guards are checked in correct order

### 404 errors in logs

- This is **normal** for new users without profiles
- Should trigger onboarding flow
- Check if error handling properly resets state

---

## Key Takeaways {#key-takeaways}

1. **`isProfileFreshFromApi`** is the critical flag that prevents stale data routing
2. **Three-layer guard system** provides defense in depth
3. **404 is valid state** - user needs to complete onboarding
4. **Backend is source of truth** - always prefer API data over cache
5. **Soft-delete auto-restore** provides graceful account recovery
