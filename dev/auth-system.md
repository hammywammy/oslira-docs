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
