# 🏗️ **OSLIRA SYSTEM ARCHITECTURE - THE COMPLETE FOUNDATION**

---

## **"The Three-Layer Fortress"**

---

### **🗄️ LAYER 1: THE VAULT (Supabase)**
**"The source of truth - immutable, secure, isolated"**

**What it holds:**
- 15 tables of structured data
- Financial audit trails (credit_ledger - never delete)
- User identity and authentication (via Supabase Auth)
- Row-level security policies (users only see their data)

**What it does:**
- ✅ Enforces data integrity (foreign keys, constraints, checks)
- ✅ Prevents invalid states (no negative balances, valid enums)
- ✅ Audits every change (triggers, timestamps)
- ✅ Isolates tenants (RLS on every query)

**What it NEVER does:**
- ❌ Makes business decisions
- ❌ Calls external APIs
- ❌ Formats data for display
- ❌ Handles complex orchestration

**Philosophy:** *"The vault knows WHAT exists, not WHY or HOW to use it."*

---

### **🔐 LAYER 1.5: THE KEYMASTER (AWS Secrets Manager)**
**"The silent guardian - never exposed, always available"**

**What it holds:**
- Supabase credentials (URL, service role key, anon key)
- Stripe keys (secret key, webhook secret)
- AI provider keys (OpenAI, Anthropic, Apify)
- Any future API credentials

**What it does:**
- ✅ Centralizes all secrets in one encrypted location
- ✅ Provides secrets on-demand to the Curator
- ✅ Never appears in code, logs, or git history
- ✅ Rotatable without code changes

**What it NEVER does:**
- ❌ Exposed to frontend (ever)
- ❌ Hardcoded in worker (never)
- ❌ Logged or printed (forbidden)

**Philosophy:** *"Secrets live in one place, accessed from one layer, visible to no one."*

---

### **🧠 LAYER 2: THE CURATOR (Cloudflare Worker)**
**"The brain - orchestrates everything, decides everything, protects everything"**

**What it does:**
- ✅ **Authentication:** Validates JWT tokens, verifies user identity
- ✅ **Authorization:** Checks permissions (even beyond RLS)
- ✅ **Orchestration:** Combines data from vault + external APIs
- ✅ **Business Logic:** Enforces rules (credit checks, plan limits)
- ✅ **Transactions:** Coordinates multi-step operations atomically
- ✅ **External APIs:** Calls Stripe, OpenAI, Apify with proper error handling
- ✅ **Webhooks:** Receives Stripe events, validates signatures, updates vault
- ✅ **Background Jobs:** Processes analyses, runs cron jobs
- ✅ **Rate Limiting:** Protects against abuse
- ✅ **Logging:** Tracks errors via Sentry
- ✅ **Monitoring:** Reports status via Uptime Robot

**What it accesses:**
- Supabase (via service role - bypasses RLS, full control)
- AWS Secrets Manager (fetches all credentials)
- Stripe API (customer/subscription management)
- OpenAI/Claude (AI analysis)
- Apify (Instagram scraping)

**What it NEVER does:**
- ❌ Returns raw database errors to frontend
- ❌ Trusts frontend input without validation
- ❌ Exposes internal IDs or structure
- ❌ Stores secrets in code

**Philosophy:** *"The curator is the only brain. It decides what the vault stores, what the gallery displays, and how external services interact. Nothing bypasses the curator."*

---

### **🎨 LAYER 3: THE GALLERY (Frontend - React/Vite)**
**"The face - beautiful, responsive, completely powerless"**

**What it does:**
- ✅ Displays data elegantly (Tailwind CSS styling)
- ✅ Captures user input (forms, clicks, interactions)
- ✅ Sends requests to curator (via fetch/axios)
- ✅ Handles loading states and errors
- ✅ Manages local UI state (not business state)

**What it accesses:**
- Supabase client (anon key only - RLS enforced)
- Curator API endpoints (for everything else)

**What it NEVER does:**
- ❌ Calls Stripe directly
- ❌ Calls AI APIs directly
- ❌ Makes business logic decisions
- ❌ Validates data (curator does this)
- ❌ Has access to service role or secrets
- ❌ Trusts its own calculations (curator is truth)

**Philosophy:** *"The gallery is a beautiful idiot. It displays what it's told, asks permission for everything, and trusts nothing it computes locally."*

---

## **🔄 THE FLOW**

```
User Action (Frontend)
    ↓
"I want to analyze @nike"
    ↓
Gallery sends request → Curator API
    ↓
Curator validates:
  1. Is user authenticated? (JWT valid?)
  2. Does user have permission? (owns this account?)
  3. Does user have credits? (check vault via service role)
  4. Is analysis already running? (check vault)
    ↓
Curator orchestrates:
  1. Fetches secrets from AWS (Apify key, OpenAI key)
  2. Calls deduct_credits() function (vault, atomic)
  3. Creates analysis record (vault, status='pending')
  4. Calls Apify (scrape Instagram profile)
  5. Calls OpenAI (analyze scraped data)
  6. Updates analysis record (vault, status='completed', results)
  7. Logs costs to ai_usage_logs (vault)
    ↓
Curator returns formatted response → Gallery
    ↓
Gallery displays: "Analysis complete! Score: 85/100"
```

---

## **🏛️ SUPPORTING SERVICES**

### **💳 Stripe (Billing Partner)**
- **Handles:** Payment processing, subscription management
- **Accessed by:** Curator only (never frontend, never vault)
- **Webhooks:** Send events to curator → Curator updates vault

### **📧 Auth (Google OAuth via Supabase)**
- **Handles:** User authentication
- **Accessed by:** Frontend (signup/login) → Creates auth.users
- **Completion:** Curator creates public.users + account + grants credits

### **🐙 GitHub (Code Repository)**
- **Handles:** Version control, CI/CD
- **Deploys:** Curator (Cloudflare Worker), Gallery (Cloudflare Pages)
- **Stores:** All code, documentation, SQL migrations

### **📊 Sentry (Error Monitoring)**
- **Handles:** Error tracking, stack traces
- **Accessed by:** Curator (logs all exceptions)
- **Alerts:** Notifies team of critical errors

### **⏰ Uptime Robot (Status Monitoring)**
- **Handles:** Service availability checks
- **Monitors:** Curator API endpoints
- **Alerts:** Notifies if curator goes down

### **🔮 Bluevine (Future: Banking & Finance)**
- **Handles:** Payouts, invoicing, financial operations
- **Accessed by:** Curator (when implemented)
- **Purpose:** Pay influencers, handle multi-party transactions

---

## **🎯 THE PYRAMID OF TRUST**

```
                    Gallery (Frontend)
                   /                 \
              "Displays"           "Requests"
                 /                       \
                /                         \
           Curator (Worker)              Curator (Worker)
          /      |        \              /      |        \
    Supabase  Stripe   AI APIs      AWS     Sentry   Uptime
     (vault) (billing) (brain)   (secrets) (errors)  (status)
```

**Trust Hierarchy:**
1. **Vault** trusts nothing (enforces constraints)
2. **Curator** trusts only AWS secrets + validated inputs
3. **Gallery** trusts nothing (all validation in curator)

---

## **💎 ARCHITECTURAL PRINCIPLES**

### **1. Single Source of Truth**
- Database is the ONLY source of truth
- No caching business logic in frontend
- No local calculations that matter

### **2. Security in Depth**
- RLS at database (prevents SQL injection bypass)
- Authorization at curator (even if RLS fails)
- Input validation at curator (never trust frontend)

### **3. Separation of Concerns**
- Vault = data integrity
- Curator = business logic + orchestration
- Gallery = user experience

### **4. Fail-Safe Defaults**
- Default deny (must explicitly grant access)
- Optimistic UI (grant credits immediately, retry on failure)
- Idempotency (webhooks, cron jobs can run multiple times safely)

### **5. Observable & Debuggable**
- Every error logged (Sentry)
- Every webhook tracked (webhook_events table)
- Every credit movement audited (credit_ledger)
- Every API call logged (ai_usage_logs)

---

## **🚀 WHY THIS ARCHITECTURE WINS LONG-TERM**

### **✅ Scalability**
- Replace gallery without touching backend
- Scale curator horizontally (serverless)
- Upgrade database without code changes

### **✅ Security**
- Secrets never in code or logs
- Multi-layer defense (RLS + curator validation)
- Service isolation (frontend can't reach vault directly)

### **✅ Maintainability**
- Each service has ONE job
- Clear boundaries (no "god objects")
- Easy to onboard new developers

### **✅ Cost Efficiency**
- Cloudflare Workers: Pay per request (scales to zero)
- Supabase: Postgres pricing (predictable)
- Serverless AI APIs: Pay per use

### **✅ Flexibility**
- Add new AI providers (just update curator)
- Switch payment processors (just update curator)
- Redesign frontend (vault + curator unchanged)

---

## **🎓 THE GOLDEN RULES**

1. **Never let the gallery touch the vault**
   - Frontend uses anon key (RLS enforced)
   - All writes go through curator

2. **Never let the vault make decisions**
   - Database enforces integrity
   - Curator handles business logic

3. **The curator is the only brain**
   - All orchestration here
   - All external API calls here
   - All secrets accessed here

4. **Secrets live in one place**
   - AWS Secrets Manager only
   - Fetched by curator at runtime
   - Never hardcoded, never logged

5. **Trust nothing, verify everything**
   - Validate inputs (curator)
   - Enforce constraints (vault)
   - Log everything (Sentry)

---

**"The vault protects data. The curator orchestrates intelligence. The gallery presents beauty. Together, they build an unbreakable system."**

⚖️ THE HONEST ANSWER
For READ operations on simple data, you're right - the worker CAN be an extra step.
But here's when it matters:

📊 WHEN TO USE EACH APPROACH
✅ DIRECT DATABASE ACCESS (Frontend → Supabase)
Use for:

Simple reads with no business logic
Data that's already in correct format
Operations where RLS is sufficient security

Examples:
javascript// ✅ GOOD: Get leads for a business profile
const { data: leads } = await supabase
  .from('leads')
  .select('*')
  .eq('business_profile_id', profileId)
  .is('deleted_at', null)
  .order('last_analyzed_at', { ascending: false });

// ✅ GOOD: Get credit balance
const { data: balance } = await supabase
  .from('credit_balances')
  .select('current_balance')
  .eq('account_id', accountId)
  .single();

// ✅ GOOD: Get user's accounts
const { data: accounts } = await supabase
  .from('accounts')
  .select(`
    *,
    account_members!inner(role),
    credit_balances(current_balance)
  `)
  .eq('account_members.user_id', userId)
  .is('deleted_at', null);
Why this works:

✅ RLS enforces security (user can only see their data)
✅ No business logic needed
✅ Faster (one less network hop)
✅ Real-time subscriptions possible
✅ Less worker usage (lower costs)


🧠 VIA WORKER (Frontend → Worker → Supabase)
Use for:

Complex queries needing multiple tables
Operations requiring business logic
Data needing formatting/transformation
Operations involving external APIs
Writes that need validation

Examples:
Example 1: Create Analysis (MUST use worker)
javascript// ❌ WRONG: Frontend directly
const { data } = await supabase.rpc('deduct_credits', { ... });
// Problem: User could manipulate parameters, bypass checks

// ✅ CORRECT: Via worker
const response = await fetch('/api/leads/analyze', {
  method: 'POST',
  body: JSON.stringify({
    leadId: 'lead_123',
    analysisType: 'deep'
  })
});

// Worker does:
// 1. Validates user owns this lead
// 2. Checks credits (deduct_credits RPC)
// 3. Calls Apify (scrape Instagram)
// 4. Calls OpenAI (analyze)
// 5. Stores results
// 6. Returns formatted response
Example 2: Get Leads with Latest Analysis Score (Could go either way)
javascript// Option A: Direct (complex query, but possible)
const { data } = await supabase
  .from('leads')
  .select(`
    *,
    latest_analysis:analyses!lead_id(
      id, overall_score, completed_at
    )
  `)
  .eq('account_id', accountId)
  .order('created_at', { foreignTable: 'analyses', ascending: false })
  .limit(1, { foreignTable: 'analyses' });

// Option B: Via worker (cleaner, adds metadata)
const response = await fetch(`/api/leads?accountId=${accountId}`);
// Returns:
// {
//   leads: [...],
//   meta: { totalCredits: 48, analysesThisMonth: 12 }
// }
Example 3: Upgrade Subscription (MUST use worker)
javascript// ❌ WRONG: Frontend directly
await supabase.from('subscriptions').update({ plan_type: 'pro' });
// Problem: Doesn't call Stripe, doesn't grant credits, breaks append-only

// ✅ CORRECT: Via worker
const response = await fetch('/api/subscriptions/upgrade', {
  method: 'POST',
  body: JSON.stringify({ planId: 'pro' })
});

// Worker does:
// 1. Validates current subscription
// 2. Calls Stripe API (create subscription)
// 3. Cancels old subscription in DB
// 4. Creates new subscription in DB
// 5. Grants pro-rated credits

🎯 MY RECOMMENDATION: HYBRID APPROACH
Architecture Pattern:
javascript// ============================================
// READS (Simple) → Direct Database
// ============================================

// Get leads list
const { data: leads } = await supabase
  .from('leads')
  .select('*')
  .eq('account_id', accountId);

// Get credit balance  
const { data: balance } = await supabase
  .from('credit_balances')
  .select('current_balance')
  .eq('account_id', accountId);

// Get business profiles
const { data: profiles } = await supabase
  .from('business_profiles')
  .select('*')
  .eq('account_id', accountId);

// ============================================
// WRITES (All) → Via Worker
// ============================================

// Create analysis
await fetch('/api/leads/analyze', { ... });

// Upgrade subscription
await fetch('/api/subscriptions/upgrade', { ... });

// Create business profile  
await fetch('/api/business-profiles', { ... });

// Delete lead (soft delete)
await fetch('/api/leads/delete', { ... });

// ============================================
// COMPLEX READS → Via Worker (Optional)
// ============================================

// Dashboard stats (aggregates multiple tables)
await fetch('/api/dashboard/stats');

// Lead with full analysis history + AI insights
await fetch(`/api/leads/${leadId}/full`);
```

---

## 📋 **DECISION MATRIX**

| Operation | Approach | Why |
|-----------|----------|-----|
| List leads | Direct DB | RLS protects, no logic needed |
| Get credit balance | Direct DB | Simple read, RLS protects |
| View analysis results | Direct DB | Already stored, just display |
| Get business profiles | Direct DB | Simple read |
| **Create analysis** | **Worker** | Deducts credits, calls APIs, complex |
| **Upgrade subscription** | **Worker** | Calls Stripe, grants credits |
| **Delete lead** | **Worker** | Soft delete logic, cascade |
| **Signup completion** | **Worker** | Creates account, grants credits, calls Stripe |
| Dashboard aggregates | Worker (optional) | Could do in DB, but cleaner in worker |

---

## 🎓 **THE GOLDEN RULE**
```
IF operation is:
  - Pure read
  - No external APIs
  - No complex validation
  - RLS is sufficient security
THEN:
  → Direct database access (frontend → Supabase)

ELSE:
  → Via worker
```

---

## 💡 **BENEFITS OF HYBRID APPROACH**

### **✅ Performance**
- Reads are faster (no worker hop)
- Less worker execution time (lower costs)
- Real-time subscriptions possible (Supabase Realtime)

### **✅ Simplicity**
- Less boilerplate for simple operations
- Easier debugging (fewer layers)
- Faster development (no endpoint needed)

### **✅ Still Secure**
- RLS enforces access control
- Worker handles all writes
- Worker handles all complex operations

### **✅ Cost Efficient**
- Worker only runs when needed
- Database does what it's good at (querying)
- No unnecessary network hops

---

## 🚨 **WHAT YOU MUST NEVER DO DIRECTLY**

Even in hybrid approach, these ALWAYS go through worker:

❌ **Never call these from frontend:**
- `supabase.rpc('deduct_credits', ...)` - Race conditions
- `supabase.from('credit_ledger').insert(...)` - Bypass audit
- Any Stripe API calls - Exposes secret keys
- Any AI API calls - Exposes API keys
- `supabase.from('subscriptions').update(...)` - Breaks append-only

---

## ✅ **UPDATED ARCHITECTURE**
```
Frontend (React)
    ├─→ Supabase Client (anon key)
    │   └─→ Simple reads (leads, profiles, balances)
    │       └─→ RLS enforces security
    │
    └─→ Cloudflare Worker (API)
        ├─→ All writes
        ├─→ Complex operations  
        ├─→ External API calls
        └─→ Business logic
            └─→ Supabase Admin (service role)
                └─→ Full database access

🎯 FINAL ANSWER
You're right - for simple reads like displaying leads, going through the worker IS an extra step.
Use direct database access for:

✅ List leads
✅ View analysis results
✅ Check credit balance
✅ Get business profiles
✅ Read-only operations with RLS

Use worker for:

✅ Create analysis (deduct credits, call APIs)
✅ Signup completion (create account, call Stripe)
✅ Upgrade subscription (Stripe + DB)
✅ Any writes
✅ Complex orchestration

This is the production-standard pattern. Companies like Notion, Linear, and Vercel use this exact hybrid approach.
