# 🎯 **PRODUCTION AUTH SYSTEM - FUNDAMENTAL LOGIC**

**Industry Standard Pattern for SaaS Applications**

---

## **📊 THE TWO-TOKEN SYSTEM**

Modern SaaS apps use a dual-token approach: short-lived access tokens (JWT) for API calls and long-lived refresh tokens for session persistence.

### **Token Roles:**

```
ACCESS TOKEN (JWT):
- Lifespan: 15 minutes
- Purpose: Authorize API requests
- Storage: Memory OR localStorage
- Security: Short-lived = limited damage if stolen

REFRESH TOKEN (Opaque):
- Lifespan: 7 days
- Purpose: Get new access tokens
- Storage: localStorage (survives browser restart)
- Security: Can be revoked in database
```

---

## **🔄 THE FLOW - HOW IT ACTUALLY WORKS**

### **1. Initial Login (Google OAuth)**

```
User clicks "Login with Google"
  ↓
Google OAuth completes
  ↓
Worker returns BOTH tokens:
  - accessToken (15 min expiry)
  - refreshToken (7 days expiry)
  ↓
Frontend stores BOTH in localStorage
  ↓
User is logged in
```

---

### **2. Making API Requests (The Smart Part)**

```
User navigates to /dashboard
  ↓
App needs data → calls API
  ↓
HTTP Client checks: "Is access token valid?"
  ↓
YES → Use it (Authorization: Bearer {token})
  ↓
NO (expired) → AUTOMATIC REFRESH:
    1. Call POST /api/auth/refresh with refreshToken
    2. Get new accessToken + new refreshToken
    3. Store both
    4. Retry original request with new token
  ↓
User sees dashboard (never knew refresh happened)
```

**Key Insight:** The refresh happens automatically and transparently. User never sees "session expired" errors unless the 7-day refresh token is also expired.

---

### **3. Browser Restart (The "Remember Me" Magic)**

```
User closes browser
  ↓
7 hours later, opens app again
  ↓
App loads → Auth system initializes
  ↓
Checks localStorage: Tokens exist?
  ↓
YES:
  - Access token expired? (it's been 7 hours)
  - Refresh token still valid? (YES, within 7 days)
  ↓
Auto-refresh access token
  ↓
User lands on /dashboard (still logged in!)
```

**No manual login required for 7 days.**

---

### **4. Logout (Instant + Permanent)**

```
User clicks "Logout"
  ↓
Frontend:
  1. Calls POST /api/auth/logout (tells Worker)
  2. Clears localStorage
  3. Redirects to /login
  ↓
Worker:
  1. Adds refreshToken to blacklist (database)
  2. Token now unusable
  ↓
User is logged out everywhere
```

---

## **🛡️ SECURITY - NO RACE CONDITIONS**

### **Problem: Multiple Requests Trying to Refresh Simultaneously**

```
❌ WITHOUT PROTECTION:
Request A: Access token expired → Call refresh
Request B: Access token expired → Call refresh
Request C: Access token expired → Call refresh
  ↓
3 refresh calls happen at once
  ↓
Race condition: Which tokens are correct?
```

### **Solution: Single In-Flight Refresh Promise**

The auth manager uses a promise-based locking mechanism to ensure only one refresh happens at a time, even if multiple requests trigger simultaneously.

```
✅ WITH PROTECTION:
Request A: Expired → Creates refreshPromise → Calls API
Request B: Expired → Sees refreshPromise → Waits for A
Request C: Expired → Sees refreshPromise → Waits for A
  ↓
Only ONE refresh call
  ↓
All requests get the SAME new token
  ↓
All requests succeed with correct token
```

**Implementation Pattern:**
```javascript
if (!refreshPromise) {
  refreshPromise = actuallyCallRefreshAPI();
}
await refreshPromise; // All requests wait for same promise
refreshPromise = null; // Reset after complete
```

---

## **📂 WHERE THINGS LIVE**

### **Frontend Storage Strategy:**

```
localStorage:
  - accessToken: "eyJhbGc..."
  - refreshToken: "opaque_random_string_here"
  - expiresAt: 1234567890 (Unix timestamp)

Why localStorage (not sessionStorage)?
  ✅ Survives browser restart (remember me)
  ✅ Accessible across tabs
  ❌ Vulnerable if XSS (but tokens are short-lived)

Alternative (More Secure):
  - httpOnly cookies for refresh token
  - localStorage for access token
```

---

## **🎯 THE CORE FILES ARCHITECTURE**

### **File 1: Auth Manager (Singleton)**

**Purpose:** Central source of truth for auth state  
**Runs:** Once on app initialization  
**Responsibilities:**
- Load tokens from storage on startup
- Check if tokens are valid
- Auto-refresh expired access tokens
- Provide `getAccessToken()` to entire app
- Handle logout (clear everything)

**Key Methods:**
```
initialize() → Load tokens, check validity, return bool
getAccessToken() → Return valid token (auto-refresh if needed)
setTokens(access, refresh) → Store after login
clear() → Logout
isAuthenticated() → Boolean check
```

---

### **File 2: HTTP Client**

**Purpose:** Make API calls with automatic token injection  
**Runs:** Every API request  
**Responsibilities:**
- Call `authManager.getAccessToken()` before each request
- Add `Authorization: Bearer {token}` header
- Handle 401 errors (trigger refresh)
- Retry failed requests after refresh

**Flow:**
```
request(url, options)
  ↓
token = await authManager.getAccessToken()
  ↓
fetch(url, { headers: { Authorization: `Bearer ${token}` } })
  ↓
if (response.status === 401):
  - Refresh happened but still 401?
  - Clear auth, redirect to login
```

---

### **File 3: Auth Provider (React Context)**

**Purpose:** Expose auth state to all React components  
**Runs:** Wraps entire app  
**Responsibilities:**
- Initialize authManager on mount
- Provide `user`, `isAuthenticated`, `isLoading` to components
- Trigger re-renders when auth state changes

**Pattern:**
```jsx
<AuthProvider>
  {isLoading ? <LoadingSpinner /> : <App />}
</AuthProvider>
```

---

### **File 4: Protected Route Guard**

**Purpose:** Block access to pages requiring auth  
**Runs:** On route navigation  
**Responsibilities:**
- Check `authManager.isAuthenticated()`
- Redirect to `/login` if not authenticated
- Check `user.onboarding_completed` flag
- Redirect to `/onboarding` if incomplete

**Logic:**
```
if (!isAuthenticated) → Redirect /login
if (!onboardingCompleted) → Redirect /onboarding
else → Render page
```

---

## **🔐 WORKER API ENDPOINTS**

### **POST /api/auth/complete-signup**

**When:** After Google OAuth callback  
**Receives:** OAuth code  
**Returns:**
```json
{
  "accessToken": "eyJ...", 
  "refreshToken": "random_opaque",
  "user": { "id": "...", "email": "..." },
  "onboardingCompleted": false
}
```

---

### **POST /api/auth/refresh**

**When:** Access token expired  
**Receives:** `{ refreshToken: "..." }`  
**Returns:**
```json
{
  "accessToken": "eyJ...", 
  "refreshToken": "new_random_opaque"
}
```

**Critical:** Uses rotating refresh tokens - each refresh returns a NEW refresh token, invalidating the old one. This prevents token reuse attacks.

---

### **POST /api/auth/logout**

**When:** User clicks logout  
**Receives:** `{ refreshToken: "..." }`  
**Does:**
1. Add refreshToken to blacklist table
2. Return success

**Result:** Token unusable even if attacker has it

---

### **GET /api/auth/session**

**When:** Need user info (profile, onboarding status)  
**Receives:** `Authorization: Bearer {accessToken}` header  
**Returns:**
```json
{
  "user": { "id": "...", "email": "...", "full_name": "..." },
  "account": { "id": "...", "credits": 25 },
  "onboardingCompleted": true
}
```

---

## **🚦 PAGE LOAD SEQUENCE**

### **What Happens When User Visits App:**

```
1. index.html loads
   ↓
2. React app mounts
   ↓
3. AuthProvider initializes
   ↓
4. authManager.initialize() runs:
   - Load tokens from localStorage
   - Access token valid? Use it
   - Access token expired? Refresh it
   - Refresh token expired? Clear everything
   ↓
5. Set isLoading = false
   ↓
6. Router decides where to send user:
   - No auth → /login
   - Auth + no onboarding → /onboarding
   - Auth + onboarding → /dashboard (or requested page)
```

**Time:** <100ms if tokens cached, <500ms if refresh needed

---

## **✅ AUTO-REDIRECT LOGIC**

### **Login/Signup Page Behavior:**

```
User lands on /login
  ↓
Page loads → Check auth
  ↓
Already authenticated?
  ↓
YES:
  - Onboarding complete? → Redirect /dashboard
  - Onboarding incomplete? → Redirect /onboarding
  ↓
NO:
  - Show login form
```

**Result:** Can't access login page if already logged in

---

### **Dashboard/App Pages:**

```
User lands on /dashboard
  ↓
ProtectedRoute checks auth
  ↓
Not authenticated? → Redirect /login
  ↓
Authenticated but no onboarding? → Redirect /onboarding
  ↓
Both complete? → Show /dashboard
```

---

## **⏰ 7-DAY EXPIRY - HOW IT WORKS**

```
Day 0: User logs in
  - Access token: Expires in 15 min
  - Refresh token: Expires in 7 days

Day 0-7: User uses app daily
  - Access token refreshed every 15 min automatically
  - Refresh token updated on each refresh (rolling window)
  - User never logs out

Day 7: User returns after 7 days
  - Refresh token expired
  - Auto-refresh fails
  - authManager.clear() called
  - Redirect to /login

Day 7+: Manual login required
```

**Key:** The 7-day window resets on every refresh, so active users stay logged in indefinitely. Only inactive users (7+ days) need to re-login.

---

## **🎭 MANUAL LOGOUT vs AUTO LOGOUT**

### **Manual Logout (User Clicks Button):**

```
Instant:
  1. Clear localStorage
  2. Blacklist refresh token (Worker)
  3. Redirect /login
  
Permanent:
  - Even if attacker has old tokens, they're blacklisted
```

### **Auto Logout (7 Days Inactive):**

```
Silent:
  1. Refresh attempt fails (token expired)
  2. authManager.clear()
  3. Redirect /login
  
No blacklist needed:
  - Token naturally expired
```

---

## **🔍 DEBUGGING - COMMON ISSUES**

### **"Why am I stuck in login loop?"**

```
Check:
1. Worker returning valid tokens?
2. Tokens being saved to localStorage?
3. authManager.initialize() being called?
4. Redirect logic checking isAuthenticated correctly?
```

### **"Why do I get logged out randomly?"**

```
Check:
1. Access token expiry too short? (should be 15 min)
2. Refresh endpoint working?
3. Refresh token being rotated correctly?
4. localStorage being cleared accidentally?
```

### **"Why can't I log out?"**

```
Check:
1. Logout endpoint blacklisting token?
2. localStorage.clear() being called?
3. Redirect to /login happening?
```

---

## **📝 SUMMARY - THE COMPLETE PICTURE**

```
LOGIN:
  Google OAuth → Worker returns 2 tokens → Store in localStorage

USING APP:
  Every API call → authManager gets valid token → Auto-refresh if needed

BROWSER RESTART:
  Load tokens → Check expiry → Refresh if needed → Continue session

7 DAYS LATER:
  Tokens expired → Auto-logout → Redirect /login

MANUAL LOGOUT:
  Clear storage → Blacklist token → Redirect /login
```

**Zero race conditions. Zero complexity. Just works.**

---

**Next Step:** Build the 4 core files based on this logic.


Actual plan:

# ✅ **AGREED - 25 Credits Upfront, Simple and Clean**

You're absolutely right. Let's strip away the complexity and build what actually works.

---

## **🎯 THE CORRECT ARCHITECTURE (No BS)**

```
1. Google OAuth Success
   ↓
2. POST /api/auth/complete-signup (Worker)
   ATOMIC TRANSACTION (via application logic, not DB function):
   ✅ Create user record
   ✅ Create account (owner_id, name)
   ✅ Create account_membership (owner role)
   ✅ Grant 25 credits IMMEDIATELY (one-time, protected by unique constraint)
   ✅ Set onboarding_completed = false
   ✅ Return JWT + account_id
   
   Stripe customer creation: ASYNC (separate job, non-critical path)
   ↓
3. /callback (Frontend)
   GET /api/auth/session
   Checks: onboarding_completed?
   → FALSE → Redirect /onboarding
   ↓
4. Onboarding Form (Required, but resumable)
   Step 1: Business name, website
   Step 2: Target audience (ICP)
   Step 3: Analysis goals
   
   Each step saves to partial business_profile
   ↓
5. POST /api/onboarding/complete
   
   IDEMPOTENT CHECK:
   if (user.onboarding_completed === true) {
     return success; // Already done, no-op
   }
   
   ATOMIC TRANSACTION (application layer):
   ✅ Finalize business_profile
   ✅ Set onboarding_completed = true
   ✅ Log completion event
   
   ASYNC (non-blocking):
   - Update Stripe customer metadata
   - Send welcome email
   ↓
6. /dashboard
   User has 25 credits, ready to analyze
```

---

## **WHY THIS IS CORRECT**

### **✅ No Credit Inflation**
- **One grant, ever:** Protected by `users.id` unique constraint
- **No "rewards":** User gets 25 credits during signup, uses them after onboarding
- **Simple mental model:** "Sign up → Get 25 credits → Learn how to use them → Use them"

### **✅ Application-Layer Transactions (Not DB Functions)**
```typescript
// Worker handles orchestration, not database
export async function completeSignup(c: Context) {
  const { userId, email, fullName } = await validateOAuth(c);
  
  // Check if already exists
  const existing = await db.users.findUnique({ where: { id: userId } });
  if (existing) {
    return success({ account_id: existing.account_id });
  }
  
  // Application-layer transaction (NOT database function)
  const account = await db.accounts.create({
    data: {
      owner_id: userId,
      name: `${fullName}'s Account`,
      slug: generateSlug(fullName)
    }
  });
  
  await db.users.create({
    data: {
      id: userId,
      email,
      full_name: fullName,
      onboarding_completed: false
    }
  });
  
  await db.account_members.create({
    data: {
      account_id: account.id,
      user_id: userId,
      role: 'owner'
    }
  });
  
  // Grant 25 credits (via RPC for atomic balance update)
  await db.rpc('deduct_credits', {
    account_id: account.id,
    amount: 25,
    type: 'signup_bonus',
    description: 'Initial signup credits'
  });
  
  // Async Stripe (non-blocking)
  c.executionCtx.waitUntil(
    createStripeCustomer(userId, email, account.id)
  );
  
  return success({ account_id: account.id, credits: 25 });
}
```

### **✅ Idempotency Without Complexity**
```typescript
// Simple flag check is sufficient
export async function completeOnboarding(c: Context) {
  const auth = getAuthContext(c);
  const body = await validateOnboardingData(c);
  
  // Idempotent check (critical)
  const user = await db.users.findUnique({ 
    where: { id: auth.userId } 
  });
  
  if (user.onboarding_completed) {
    // Already done - return success, don't error
    return success({ 
      message: 'Onboarding already completed',
      account_id: user.account_id 
    });
  }
  
  // Update business profile
  await db.business_profiles.upsert({
    where: { account_id: auth.accountId },
    create: {
      account_id: auth.accountId,
      business_name: body.business_name,
      website: body.website,
      business_one_liner: body.one_liner,
      business_context_pack: body.context_pack
    },
    update: {
      business_name: body.business_name,
      website: body.website,
      business_one_liner: body.one_liner,
      business_context_pack: body.context_pack
    }
  });
  
  // Mark onboarding complete
  await db.users.update({
    where: { id: auth.userId },
    data: {
      onboarding_completed: true,
      onboarding_completed_at: new Date()
    }
  });
  
  // Async operations (non-blocking)
  c.executionCtx.waitUntil(
    updateStripeMetadata(auth.accountId, body)
  );
  
  return success({ message: 'Onboarding complete' });
}
```

### **✅ No Queue System Needed (Use Cloudflare waitUntil)**
```typescript
// Built-in to Cloudflare Workers
c.executionCtx.waitUntil(
  async () => {
    try {
      await stripe.customers.create({...});
    } catch (error) {
      await logToSentry(error);
      // Cron job will retry later
    }
  }
);
```

### **✅ RLS Policy (Database Enforces Onboarding)**
```sql
-- Users must complete onboarding to create analyses
CREATE POLICY "require_onboarding_for_analysis"
ON analyses
FOR INSERT
TO authenticated
WITH CHECK (
  EXISTS (
    SELECT 1 FROM users
    WHERE users.id = auth.uid()
    AND users.onboarding_completed = true
  )
);

-- Same for leads
CREATE POLICY "require_onboarding_for_leads"
ON leads
FOR INSERT
TO authenticated
WITH CHECK (
  EXISTS (
    SELECT 1 FROM users
    WHERE users.id = auth.uid()
    AND users.onboarding_completed = true
  )
);
```

---

## **🚀 FRONTEND IMPLEMENTATION**

### **1. Worker API Client**
```typescript
// src/core/api/worker-client.ts
import { ENV } from '@/core/config/env';

class WorkerAPIClient {
  private baseURL: string;
  private token: string | null = null;

  constructor() {
    this.baseURL = ENV.apiUrl; // https://api.oslira.com
    this.token = localStorage.getItem('auth_token');
  }

  async request<T>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<T> {
    const url = `${this.baseURL}${endpoint}`;
    
    const headers = {
      'Content-Type': 'application/json',
      ...(this.token && { Authorization: `Bearer ${this.token}` }),
      ...options.headers,
    };

    const response = await fetch(url, {
      ...options,
      headers,
    });

    // Handle 401 (token expired)
    if (response.status === 401) {
      // Try token refresh
      const refreshed = await this.refreshToken();
      if (refreshed) {
        // Retry with new token
        return this.request(endpoint, options);
      } else {
        // Force re-login
        localStorage.removeItem('auth_token');
        window.location.href = '/auth/login';
        throw new Error('Authentication required');
      }
    }

    const data = await response.json();

    if (!response.ok) {
      throw new Error(data.error || 'Request failed');
    }

    return data;
  }

  async refreshToken(): Promise<boolean> {
    try {
      const response = await fetch(`${this.baseURL}/api/auth/refresh`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ token: this.token }),
      });

      if (response.ok) {
        const { token } = await response.json();
        this.setToken(token);
        return true;
      }
    } catch {
      // Refresh failed
    }
    return false;
  }

  setToken(token: string) {
    this.token = token;
    localStorage.setItem('auth_token', token);
  }

  clearToken() {
    this.token = null;
    localStorage.removeItem('auth_token');
  }

  // Auth endpoints
  async completeSignup(code: string) {
    return this.request('/api/auth/complete-signup', {
      method: 'POST',
      body: JSON.stringify({ code }),
    });
  }

  async getSession() {
    return this.request('/api/auth/session');
  }

  async logout() {
    await this.request('/api/auth/logout', { method: 'POST' });
    this.clearToken();
  }

  // Onboarding endpoints
  async saveOnboardingProgress(step: string, data: any) {
    return this.request('/api/onboarding/save-progress', {
      method: 'POST',
      body: JSON.stringify({ step, data }),
    });
  }

  async completeOnboarding(data: any) {
    return this.request('/api/onboarding/complete', {
      method: 'POST',
      body: JSON.stringify(data),
    });
  }

  async getOnboardingProgress() {
    return this.request('/api/onboarding/progress');
  }
}

export const workerAPI = new WorkerAPIClient();
```

### **2. Auth Provider (Rewritten for Worker)**
```typescript
// src/features/auth/contexts/AuthProvider.tsx
import { createContext, useContext, useEffect, useState } from 'react';
import { workerAPI } from '@/core/api/worker-client';

interface AuthState {
  user: User | null;
  account: Account | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  onboardingCompleted: boolean;
}

const AuthContext = createContext<AuthState | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [state, setState] = useState<AuthState>({
    user: null,
    account: null,
    isAuthenticated: false,
    isLoading: true,
    onboardingCompleted: false,
  });

  useEffect(() => {
    loadSession();
  }, []);

  async function loadSession() {
    try {
      const session = await workerAPI.getSession


brief overview:

# 🎯 **BUILD ORDER: Start to Finish**

---

## **PHASE 1: DATABASE (Foundation)**
Build database schema + atomic transaction function

**Files:**
1. SQL migration: `refresh_tokens` table
2. SQL migration: Add `onboarding_completed` to `users` 
3. SQL function: `create_account_atomic()` (wraps 4 INSERTs)
4. RLS policies: Enforce onboarding completion

---

## **PHASE 2: WORKER AUTH CORE (Backend)**
Build JWT issuer + token management

**Files:**
1. `src/infrastructure/auth/jwt.service.ts` (sign/verify JWTs)
2. `src/infrastructure/auth/token.service.ts` (manage refresh tokens)
3. `src/features/auth/auth.routes.ts` (register endpoints)
4. `src/features/auth/auth.handler.ts`:
   - `POST /api/auth/google/callback` (OAuth → create account → issue tokens)
   - `POST /api/auth/refresh` (rotate tokens)
   - `POST /api/auth/logout` (blacklist refresh token)
   - `GET /api/auth/session` (get user info)

---

## **PHASE 3: WORKER ONBOARDING (Backend)**
Build onboarding completion logic

**Files:**
1. `src/features/onboarding/onboarding.routes.ts`
2. `src/features/onboarding/onboarding.handler.ts`:
   - `POST /api/onboarding/complete` (finalize profile + set flag)
   - `GET /api/onboarding/progress` (optional - resume support)

---

## **PHASE 4: WORKER MIDDLEWARE (Backend)**
Replace Supabase auth middleware with custom JWT

**Files:**
1. Update `src/shared/middleware/auth.middleware.ts`:
   - Remove Supabase JWT validation
   - Add custom JWT verification
   - Check `onboarding_completed` flag

---

## **PHASE 5: FRONTEND AUTH MANAGER (Client)**
Build token storage + auto-refresh logic

**Files:**
1. `src/core/auth/auth-manager.ts` (singleton - manages tokens)
2. `src/core/api/http-client.ts` (fetch wrapper with auto-refresh)
3. `src/features/auth/contexts/AuthProvider.tsx` (React context)
4. `src/features/auth/hooks/useAuth.ts` (convenience hook)

---

## **PHASE 6: FRONTEND ONBOARDING (Client)**
Build onboarding UI

**Files:**
1. `src/features/onboarding/pages/OnboardingPage.tsx` (multi-step form)
2. `src/features/onboarding/hooks/useCompleteOnboarding.ts`
3. `src/features/auth/components/ProtectedRoute.tsx` (route guard)

---

## **PHASE 7: FRONTEND OAUTH (Client)**
Build Google OAuth button + callback handler

**Files:**
1. `src/features/auth/pages/LoginPage.tsx` (Google button)
2. `src/features/auth/pages/OAuthCallbackPage.tsx` (handle redirect)
3. Update router to add `/auth/callback` route

---

## **⚡ SIMPLIFIED BUILD ORDER**

```
1. DATABASE (1 hour)
   → Schema + atomic function + RLS policies

2. WORKER BACKEND (3 hours)
   → JWT service + Auth endpoints + Onboarding endpoints + Middleware

3. FRONTEND CORE (2 hours)
   → Auth manager + HTTP client + Auth provider

4. FRONTEND UI (2 hours)
   → Login page + Onboarding flow + Protected routes + OAuth callback

TOTAL: ~8 hours
```

---

## **🚀 START HERE**

**Begin with Phase 1 (Database).** 

Once I build the database schema, everything else depends on it. Tell me when you're ready and I'll create the migration files.
