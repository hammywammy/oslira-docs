

**📦 OSLIRA DATABASE V2 - COMPLETE IMPLEMENTATION GUIDE**

# 🔧 PHASE 0: CRITICAL FIXES & DECISIONS

**Add this section to the FRONT of your database v2 document**

---

## **📋 QUICK SUMMARY**

This Phase 0 patch resolves **4 blocking issues** and **6 architectural decisions** that must be finalized before table creation.

### **What Changed:**
1. ✅ Fixed Instagram username uniqueness (multi-business support)
2. ✅ Added credit deduction function (prevents race conditions)
3. ✅ Added account-level suspension
4. ✅ Added analysis requester tracking
5. ✅ Added slug generation function
6. ✅ Added minimal invoice tracking table
7. ✅ Defined signup flow pattern
8. ✅ Removed redundant `is_active` flags
9. ✅ Updated constraints for correct behavior
10. ✅ Added essential RLS protection

---

## **🎯 ARCHITECTURAL DECISIONS LOCKED**

### **1. Account Creation Flow**
**Pattern:** Cloudflare Worker Handler (explicit, not database trigger)

**Why:** 
- Stripe API integration happens during signup
- Explicit control for debugging
- Can customize per signup source (referral codes, promo campaigns)

**Implementation:** See "Signup Flow" section below

---

### **2. Account Naming & Slugs**
```javascript
// On signup:
const fullName = user.user_metadata.full_name || user.email.split('@')[0];
const accountName = `${fullName}'s Account`;  // "Hamza's Account"
const slug = generateSlug(fullName);          // "hamza" or "hamzawil"

// URL structure: app.oslira.com/accounts/hamzawil/dashboard
```

**Slug generation:** See function in "Essential Functions" section

---

### **3. Stripe Integration for ALL Plans**
```javascript
// FREE plans create Stripe customer (no subscription object)
const customer = await stripe.customers.create({ email, name });

await db.insert('subscriptions', {
  plan_type: 'free',
  stripe_customer_id: customer.id,        // ✅ Always present
  stripe_subscription_id: null,           // ✅ NULL for free
  status: 'active',                       // ✅ Immediately active
  current_period_start: NOW(),
  current_period_end: NOW() + 30 days     // ✅ Free has periods too
});
```

**Monthly cron grants credits to ALL plans** (free + paid, same logic)

---

### **4. Multi-Business Lead Separation**
```sql
-- Each business has independent CRM
-- Same Instagram account can be analyzed for different businesses
UNIQUE (account_id, business_profile_id, instagram_username)

-- Example:
-- Agency analyzes @nike for "Campaign A" → Lead #1
-- Agency analyzes @nike for "Campaign B" → Lead #2 ✅ ALLOWED
```

---

### **5. Re-Analysis Always Allowed**
```sql
-- No constraint on analyses.lead_id
-- Can create multiple analyses for same lead

-- Use case: Track influencer growth over time
-- Week 1: Score 75
-- Week 4: Score 82
```

---

### **6. Soft Deletes Only (No is_active)**
```sql
-- REMOVED from schema:
-- business_profiles.is_active boolean

-- ONLY USE:
-- business_profiles.deleted_at timestamptz

-- Query pattern:
WHERE deleted_at IS NULL  -- Active records
```

---

### **7. Team Permissions (Deferred to Phase 2)**
```sql
-- For now: All account_members have equal access
-- RLS: account_id IN (SELECT auth.user_account_ids())
-- No role-based restrictions yet
```

---

### **8. Admin Global Bypass**
```sql
-- users.is_admin = true bypasses ALL RLS policies
-- Use for support, debugging, admin dashboard
```

---

### **9. Invoice Tracking**
**Decision:** Keep minimal 6-column table

**Why:** 
- Support tickets resolve faster (instant query vs Stripe API call)
- Revenue dashboard works without Stripe exports
- Webhook durability (record persists even if processing fails)

---

## **🔧 SCHEMA CORRECTIONS**

### **CORRECTION #1: Instagram Username Constraint**

**FIND THIS in your doc (around line 750):**
```sql
leads (
  instagram_username text UNIQUE NOT NULL,  -- ❌ WRONG
```

**REPLACE WITH:**
```sql
leads (
  instagram_username text NOT NULL,  -- Removed UNIQUE
```

**ADD THIS constraint at table end:**
```sql
-- Allow same Instagram account for different businesses
CONSTRAINT unique_username_per_business 
  UNIQUE (account_id, business_profile_id, instagram_username)
```

**ADD THIS index (for global lookups):**
```sql
CREATE INDEX idx_leads_username_lookup 
ON leads(instagram_username) 
WHERE deleted_at IS NULL;
```

---

### **CORRECTION #2: Add Account Suspension**

**FIND THIS in your doc (accounts table, around line 200):**
```sql
accounts (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  name text NOT NULL,
  slug text UNIQUE,
  deleted_at timestamptz,
  created_at timestamptz DEFAULT NOW(),
  updated_at timestamptz DEFAULT NOW()
)
```

**ADD THESE columns before created_at:**
```sql
  is_suspended boolean DEFAULT false,
  suspended_at timestamptz,
  suspended_reason text,
  suspended_by uuid REFERENCES users(id),
```

---

### **CORRECTION #3: Add Analysis Requester**

**FIND THIS in your doc (analyses table, around line 800):**
```sql
analyses (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id uuid NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
  account_id uuid NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
  business_profile_id uuid REFERENCES business_profiles(id) ON DELETE SET NULL,
```

**ADD THIS column after business_profile_id:**
```sql
  requested_by uuid REFERENCES users(id),
```

**ADD THIS index later in doc:**
```sql
CREATE INDEX idx_analyses_requester 
ON analyses(requested_by, created_at DESC);
```

---

### **CORRECTION #4: Remove is_active from business_profiles**

**FIND THIS in your doc (business_profiles table, around line 700):**
```sql
  is_active boolean DEFAULT true,  -- ❌ DELETE THIS LINE
  deleted_at timestamptz,
```

**REMOVE the is_active line entirely**

---

### **CORRECTION #5: Add Minimal Invoice Table**

**ADD THIS table in "Layer 2: Billing Foundation" (after webhook_events):**
```sql
CREATE TABLE stripe_invoices (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
  stripe_invoice_id text UNIQUE NOT NULL,
  amount_cents integer NOT NULL,
  status text NOT NULL CHECK (status IN ('paid', 'open', 'void', 'uncollectible')),
  paid_at timestamptz,
  created_at timestamptz DEFAULT NOW()
);

CREATE INDEX idx_invoices_account ON stripe_invoices(account_id, created_at DESC);
CREATE INDEX idx_invoices_status ON stripe_invoices(status, paid_at DESC);
```

---

## **🛠️ ESSENTIAL FUNCTIONS**

### **FUNCTION #1: Credit Deduction (Prevents Race Conditions)**

**Add this to Phase 1 in your doc:**

```sql
CREATE OR REPLACE FUNCTION deduct_credits(
  p_account_id uuid,
  p_amount integer,  -- Negative for deductions, positive for grants
  p_transaction_type text,
  p_description text,
  p_metadata jsonb DEFAULT '{}'::jsonb
) RETURNS uuid AS $$
DECLARE
  v_current_balance integer;
  v_new_balance integer;
  v_transaction_id uuid;
BEGIN
  -- Lock and get current balance atomically
  SELECT COALESCE(current_balance, 0) INTO v_current_balance
  FROM credit_balances
  WHERE account_id = p_account_id
  FOR UPDATE;  -- Prevents concurrent modifications
  
  -- If no balance record exists, initialize to 0
  IF NOT FOUND THEN
    INSERT INTO credit_balances (account_id, current_balance, last_transaction_at)
    VALUES (p_account_id, 0, NOW());
    v_current_balance := 0;
  END IF;
  
  -- Calculate new balance
  v_new_balance := v_current_balance + p_amount;
  
  -- Prevent overdraft
  IF v_new_balance < 0 THEN
    RAISE EXCEPTION 'Insufficient credits: % available, % required', 
      v_current_balance, ABS(p_amount);
  END IF;
  
  -- Insert transaction with calculated balance
  INSERT INTO credit_ledger (
    account_id, 
    amount, 
    balance_after, 
    transaction_type, 
    description, 
    metadata,
    created_at
  ) VALUES (
    p_account_id, 
    p_amount, 
    v_new_balance,
    p_transaction_type, 
    p_description, 
    p_metadata,
    NOW()
  ) RETURNING id INTO v_transaction_id;
  
  -- Update materialized balance
  UPDATE credit_balances
  SET current_balance = v_new_balance,
      last_transaction_at = NOW(),
      updated_at = NOW()
  WHERE account_id = p_account_id;
  
  RETURN v_transaction_id;
END;
$$ LANGUAGE plpgsql;
```

**Usage in application:**
```javascript
// Deduct 2 credits for analysis
const txId = await db.rpc('deduct_credits', {
  p_account_id: accountId,
  p_amount: -2,  // Negative = deduction
  p_transaction_type: 'analysis',
  p_description: `Deep analysis of @${username}`,
  p_metadata: { analysis_id: analysisId, lead_id: leadId }
});
// Throws exception if insufficient credits
```

---

### **FUNCTION #2: Slug Generation**

**Add this to Phase 1 in your doc:**

```sql
CREATE OR REPLACE FUNCTION generate_slug(input_text text)
RETURNS text AS $$
DECLARE
  base_slug text;
  final_slug text;
  counter integer := 0;
BEGIN
  -- Convert to lowercase, replace non-alphanumeric with hyphens
  base_slug := lower(regexp_replace(input_text, '[^a-zA-Z0-9]+', '-', 'g'));
  
  -- Remove leading/trailing hyphens
  base_slug := trim(both '-' from base_slug);
  
  -- Truncate to 50 characters
  base_slug := substring(base_slug from 1 for 50);
  
  -- Ensure uniqueness
  final_slug := base_slug;
  
  WHILE EXISTS (SELECT 1 FROM accounts WHERE slug = final_slug) LOOP
    counter := counter + 1;
    final_slug := base_slug || '-' || counter;
  END LOOP;
  
  RETURN final_slug;
END;
$$ LANGUAGE plpgsql;
```

**Usage in application:**
```javascript
const slug = await db.rpc('generate_slug', { 
  input_text: fullName 
});
// "Hamza Williams" → "hamza-williams"
// If exists → "hamza-williams-2"
```

---

## **📝 SIGNUP FLOW IMPLEMENTATION**

**Add this to your Cloudflare Worker:**

```javascript
/**
 * POST /api/auth/complete-signup
 * Called after Supabase auth.signUp() succeeds
 */
export async function completeSignup(c) {
  const { userId, email, fullName } = await c.req.json();
  
  try {
    // 1. Create Stripe customer
    const customer = await stripe.customers.create({
      email,
      name: fullName,
      metadata: { 
        user_id: userId,
        source: 'oslira_signup'
      }
    });
    
    // 2. Generate unique slug
    const slug = await db.rpc('generate_slug', { 
      input_text: fullName 
    });
    
    // 3. Create account
    const account = await db.query(`
      INSERT INTO accounts (owner_id, name, slug)
      VALUES ($1, $2, $3)
      RETURNING id
    `, [userId, `${fullName}'s Account`, slug]);
    
    const accountId = account.rows[0].id;
    
    // 4. Add user to account_members
    await db.query(`
      INSERT INTO account_members (account_id, user_id, role)
      VALUES ($1, $2, 'owner')
    `, [accountId, userId]);
    
    // 5. Create free subscription
    const periodStart = new Date();
    const periodEnd = new Date(Date.now() + 30*24*60*60*1000); // 30 days
    
    await db.query(`
      INSERT INTO subscriptions (
        account_id, 
        plan_type, 
        price_cents,
        stripe_customer_id, 
        stripe_subscription_id,
        status,
        current_period_start,
        current_period_end
      ) VALUES ($1, 'free', 0, $2, NULL, 'active', $3, $4)
    `, [accountId, customer.id, periodStart, periodEnd]);
    
    // 6. Grant 25 welcome credits
    await db.rpc('deduct_credits', {
      p_account_id: accountId,
      p_amount: 25,  // Positive = grant
      p_transaction_type: 'signup_bonus',
      p_description: 'Welcome to Oslira!'
    });
    
    return c.json({
      success: true,
      account: {
        id: accountId,
        slug,
        credits: 25
      },
      stripe_customer_id: customer.id
    });
    
  } catch (error) {
    console.error('Signup failed:', error);
    return c.json({ success: false, error: error.message }, 500);
  }
}
```

**Frontend flow:**
```javascript
// 1. User fills signup form
const { data, error } = await supabase.auth.signUp({
  email,
  password,
  options: {
    data: { full_name: fullName }
  }
});

// 2. Call backend to complete setup
const response = await fetch('/api/auth/complete-signup', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: data.user.id,
    email: data.user.email,
    fullName
  })
});

// 3. Redirect to onboarding
window.location.href = `/accounts/${response.account.slug}/onboarding`;
```

---

## **🔒 UPDATED RLS POLICIES**

**REPLACE your RLS helper function with this:**

```sql
-- Helper function for account access
CREATE OR REPLACE FUNCTION auth.user_account_ids()
RETURNS SETOF uuid
LANGUAGE sql
STABLE
SECURITY DEFINER
AS $$
  SELECT am.account_id 
  FROM account_members am
  JOIN accounts a ON a.id = am.account_id
  WHERE am.user_id = auth.uid()
    AND a.deleted_at IS NULL
    AND a.is_suspended = false
$$;
```

**ADD RLS to account_members table:**

```sql
ALTER TABLE account_members ENABLE ROW LEVEL SECURITY;

-- Only account owners can manage members
CREATE POLICY "owners_manage_members" ON account_members
FOR ALL
USING (
  account_id IN (
    SELECT id FROM accounts 
    WHERE owner_id = auth.uid() 
    AND deleted_at IS NULL
  )
);

-- Service role bypass
CREATE POLICY "service_role_bypass" ON account_members
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

**UPDATE all existing RLS policies to include admin bypass:**

```sql
-- Example for leads table
CREATE POLICY "account_access" ON leads
FOR ALL
USING (
  -- Admin users see everything
  EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND is_admin = true)
  OR
  -- Regular users see their account's data
  (account_id IN (SELECT auth.user_account_ids()) AND deleted_at IS NULL)
);
```

**Apply this pattern to:**
- leads
- analyses
- business_profiles
- account_usage_summary
- credit_ledger (read-only)

---

## **📊 WEBHOOK HANDLERS**

**ADD invoice tracking to webhook handlers:**

```javascript
/**
 * Handle invoice.paid webhook
 */
export async function handleInvoicePaid(invoice) {
  // Get account from Stripe customer ID
  const account = await db.query(`
    SELECT account_id FROM subscriptions 
    WHERE stripe_customer_id = $1 
    LIMIT 1
  `, [invoice.customer]);
  
  if (!account.rows.length) {
    console.error('No account found for customer:', invoice.customer);
    return;
  }
  
  const accountId = account.rows[0].account_id;
  
  // Store invoice record
  await db.query(`
    INSERT INTO stripe_invoices (
      account_id,
      stripe_invoice_id,
      amount_cents,
      status,
      paid_at
    ) VALUES ($1, $2, $3, 'paid', $4)
    ON CONFLICT (stripe_invoice_id) DO NOTHING
  `, [
    accountId,
    invoice.id,
    invoice.amount_paid,
    new Date(invoice.status_transitions.paid_at * 1000)
  ]);
  
  console.log('Invoice recorded:', invoice.id);
}

/**
 * Handle invoice.payment_failed webhook
 */
export async function handleInvoiceFailed(invoice) {
  const account = await db.query(`
    SELECT account_id FROM subscriptions 
    WHERE stripe_customer_id = $1 
    LIMIT 1
  `, [invoice.customer]);
  
  if (!account.rows.length) return;
  
  const accountId = account.rows[0].account_id;
  
  // Store failed invoice
  await db.query(`
    INSERT INTO stripe_invoices (
      account_id,
      stripe_invoice_id,
      amount_cents,
      status
    ) VALUES ($1, $2, $3, 'uncollectible')
    ON CONFLICT (stripe_invoice_id) DO NOTHING
  `, [accountId, invoice.id, invoice.amount_due]);
  
  // Update subscription status
  await db.query(`
    UPDATE subscriptions
    SET status = 'past_due'
    WHERE stripe_customer_id = $1
  `, [invoice.customer]);
}
```

---

## **✅ PHASE 0 CHECKLIST**

Before proceeding to table creation:

- [ ] Applied all 5 schema corrections to your doc
- [ ] Added `deduct_credits()` function to Phase 1
- [ ] Added `generate_slug()` function to Phase 1
- [ ] Added `stripe_invoices` table definition
- [ ] Updated RLS helper function with suspension check
- [ ] Added RLS policies to `account_members`
- [ ] Updated all table RLS policies with admin bypass
- [ ] Implemented signup flow in Cloudflare Worker
- [ ] Added invoice webhook handlers
- [ ] Removed all references to `is_active` columns

---

## **🚀 WHAT'S NEXT**

### **Immediate (Today):**
1. Review this Phase 0 document
2. Apply corrections to your master doc
3. Validate no conflicts

### **Tomorrow (Deploy):**
1. Run Phase 1 SQL (table creation + constraints)
2. Deploy `deduct_credits()` and `generate_slug()` functions
3. Test signup flow end-to-end
4. Verify RLS policies work

### **Next Week:**
1. Add remaining indexes (Phase 2)
2. Add helper RPC functions
3. Implement cron jobs
4. Load test credit deduction under concurrency

---

**STATUS: Phase 0 Complete ✅**

Your schema is now production-ready for implementation.




---

## **🎯 PROJECT OVERVIEW**

**Product:** Instagram lead analysis platform (B2B SaaS)  
**Tech Stack:** Supabase (PostgreSQL), Cloudflare Workers, Stripe, OpenAI/Claude/Apify  
**Database State:** Nearly empty (< 2 MB test data) - perfect for clean V2 rebuild  
**Goal:** Production-ready schema with zero future refactoring needed

---

## **✅ LOCKED ARCHITECTURAL DECISIONS**

### **1. BILLING MODEL: Account-Based (Not User-Based)**

**Core Principle:** Credits owned by accounts (not individual users)

```
HIERARCHY:
└── users (identity)
    └── account_members (junction table)
        └── accounts (THE BILLABLE ENTITY)
            ├── subscriptions (Stripe lives here)
            ├── credit_ledger (append-only source of truth)
            └── business_profiles (multiple businesses, shared credits)
                └── leads & analyses
```

**Why Account-Based:**
- One user can own 5 businesses under ONE account with SHARED credit pool
- Enables team collaboration (multiple users, one account, shared credits)
- Industry standard (Notion, Slack, Figma all use this model)
- Easier to add team members later without schema changes

---

### **2. CREDIT SYSTEM: Dual-Table Architecture**

**TWO separate tables with different purposes:**

#### **A) `credit_ledger` (Append-Only Audit Log)**
- **Purpose:** Financial audit trail, never UPDATE only INSERT
- **Stores:** Every credit movement (+500 renewal, -2 analysis)
- **Source of Truth:** Balance = `SUM(amount)` (will materialize later for performance)
- **Retention:** Forever (legal/accounting requirement)

#### **B) `account_usage_summary` (Monthly Rollup)**
- **Purpose:** Performance optimization for dashboard queries
- **Stores:** Aggregated stats per account per month
- **Updated:** Real-time during month, frozen at month-end
- **Retention:** Forever (historical trends)

**Why Both:**
- `credit_ledger` = trustworthy audit (legal requirement)
- `account_usage_summary` = fast queries (don't scan millions of rows)

---

### **3. AI COST TRACKING: Separate from Credit Accounting**

**CRITICAL SEPARATION:** Billing ≠ Performance Monitoring

#### **A) `credit_ledger` (User Charges)**
- Tracks what user PAID (2 credits = -2 row)
- No AI provider details here

#### **B) `ai_usage_logs` (Actual API Costs)**
- Tracks every OpenAI/Claude/Apify call separately
- 1 analysis = 3-8 rows (scrape + AI analysis + message generation)
- Stores: provider, model, tokens, cost_cents, duration_ms
- **Retention:** 90 days only (archived after)

#### **C) `platform_metrics_daily` (Admin Dashboard)**
- Daily rollup of platform-wide stats
- Aggregates from `ai_usage_logs` + other sources
- Fast admin queries without scanning millions of rows

**Flow Example:**
```
User runs deep analysis:
1. Apify scrape → INSERT ai_usage_logs (cost: 5¢, duration: 4.5s)
2. OpenAI analysis → INSERT ai_usage_logs (cost: 12¢, tokens: 2500)
3. Claude message → INSERT ai_usage_logs (cost: 8¢, tokens: 1800)
4. Total cost: 25¢
5. Charge user: INSERT credit_ledger (amount: -2, metadata: {actual_cost_cents: 25})
6. Update rollups: account_usage_summary, platform_metrics_daily
```

---

### **4. SUBSCRIPTIONS: Append-Only (Never UPDATE plan_type)**

**Rule:** NEVER mutate existing subscription rows

```sql
-- ❌ WRONG
UPDATE subscriptions SET plan_type = 'pro' WHERE id = 'xxx';

-- ✅ CORRECT
-- 1. Cancel old subscription
UPDATE subscriptions 
SET status = 'canceled', canceled_at = NOW() 
WHERE account_id = 'acc_123' AND status = 'active';

-- 2. Insert new subscription
INSERT INTO subscriptions (account_id, plan_type, ...) 
VALUES ('acc_123', 'pro', ...);
```

**Why:** Preserves full subscription history (free → pro → enterprise audit trail)

---

### **5. STRIPE INTEGRATION: Optimistic Credit Grant**

**Flow:**
```
1. User signs up → Supabase auth.users created
   ↓
2. IMMEDIATE (optimistic, no waiting):
   - INSERT accounts
   - INSERT subscriptions (status = 'pending_stripe_confirmation')
   - INSERT credit_ledger (+25 credits)
   - User can START using the app immediately
   ↓
3. BACKGROUND (async, 2-5 seconds):
   - Cloudflare Worker calls Stripe API
   - Creates customer + subscription
   ↓
4. WEBHOOK (when Stripe confirms):
   - UPDATE subscriptions SET stripe_customer_id, stripe_subscription_id, status = 'active'
   ↓
5. IF STRIPE FAILS (network error, etc.):
   - Cron job retries every 5 minutes
   - User KEEPS their 25 credits (optimistic grant stands)
```

**Critical Fields:**
```sql
subscriptions (
  stripe_customer_id text,  -- ← TEXT not UUID (Stripe returns "cus_abc123")
  stripe_subscription_id text,  -- ← TEXT not UUID (Stripe returns "sub_xyz789")
  status text  -- 'pending_stripe_confirmation' → 'active'
)
```

---

### **6. WEBHOOK IDEMPOTENCY: Prevent Duplicate Processing**

**Table:** `webhook_events`

```sql
CREATE TABLE webhook_events (
  stripe_event_id text PRIMARY KEY,  -- "evt_1A2B3C..."
  event_type text NOT NULL,
  account_id uuid,
  processed_at timestamptz DEFAULT NOW(),
  payload jsonb
)
```

**Handler Pattern:**
```javascript
// ALWAYS check FIRST before processing
const existing = await db.query(
  'SELECT 1 FROM webhook_events WHERE stripe_event_id = $1',
  [event.id]
);

if (existing.rows.length > 0) {
  return { received: true, message: 'already_processed' };
}

// Process event...
// Then record it
await db.insert('webhook_events', {...});
```

**Why Critical:** Stripe retries webhooks on timeout. Without this, user gets double credits.

---

### **7. LEADS + ANALYSES: Separate Tables, JSONB Inline**

**Design Decision:** Keep separate (not denormalized)

```sql
leads (
  -- Profile data (PATCHED on each re-analysis)
  instagram_username text UNIQUE,
  follower_count integer,
  bio text,
  -- etc.
  
  -- Metadata
  first_analyzed_at timestamptz,
  last_analyzed_at timestamptz  -- ← Updated on re-analysis
)

analyses (
  lead_id uuid FK → leads,
  
  -- AI response INLINE (no separate payloads table)
  ai_response jsonb,  -- Full response stored here
  
  -- Extracted for indexing
  overall_score integer,
  niche_fit_score integer,
  
  -- Cost tracking
  credits_charged integer,
  total_cost_cents integer
)
```

**Why No Denormalization:**
- ❌ Don't store `latest_score` in leads table (sync bugs)
- ✅ Use indexes + LATERAL joins for fast queries
- ✅ Supports historical analysis tracking (UI feature later)

---

### **8. BUSINESS PROFILES: Minimal Schema, JSONB for Flexibility**

**User-Editable Fields:**
```sql
business_profiles (
  business_name text,  -- Displayed in UI everywhere
  website text,  -- User can edit directly
  business_one_liner text,  -- Displayed in dashboard greeting
  
  -- Everything else in JSONB
  business_context_pack jsonb  -- {niche, target_audience, value_prop, ...}
)
```

**Why JSONB:**
- Onboarding collects 15+ fields (niche, audience, problems, etc.)
- AI generates `business_context_pack` from these
- User rarely edits (set-and-forget)
- If editing needed: "Regenerate from Scratch" button shows onboarding form again

**Settings UI:**
```
┌─ Business Profile ────────────────┐
│ Name: [Oslira                  ]  │ ← Direct edit
│ Website: [oslira.com           ]  │ ← Direct edit
│ One-liner: [We help copywriters...]│ ← Direct edit
│                                   │
│ [Regenerate Business Context] ← Opens onboarding form
└───────────────────────────────────┘
```

---

### **9. SOFT DELETES: 30-Day Recovery Window**

**Tables with `deleted_at`:**
- accounts
- business_profiles
- leads
- analyses

**Query Pattern:**
```sql
-- ALWAYS filter out deleted records
SELECT * FROM leads 
WHERE account_id = $1 
  AND deleted_at IS NULL;
```

**Cleanup Cron (Daily):**
```sql
-- Hard delete after 30 days
DELETE FROM leads WHERE deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM analyses WHERE deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM business_profiles WHERE deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM accounts WHERE deleted_at < NOW() - INTERVAL '30 days';
```

**Partial Indexes (Performance):**
```sql
-- Only index active records
CREATE INDEX idx_leads_active 
ON leads(account_id, last_analyzed_at DESC) 
WHERE deleted_at IS NULL;
```

---

### **10. FOREIGN KEY CASCADE RULES**

**Strategy:** Preserve audit trail, cascade only where data is meaningless without parent

```sql
-- CASCADE (child meaningless without parent)
analyses.lead_id → leads(id) ON DELETE CASCADE
account_members.account_id → accounts(id) ON DELETE CASCADE

-- SET NULL (preserve audit trail)
credit_ledger.analysis_id → analyses(id) ON DELETE SET NULL
credit_ledger.lead_id → leads(id) ON DELETE SET NULL
analyses.business_profile_id → business_profiles(id) ON DELETE SET NULL

-- RESTRICT (prevent accidental hard deletes, use soft delete)
accounts.id ← all child tables ON DELETE RESTRICT
```

**Example:**
```
User soft-deletes lead:
  UPDATE leads SET deleted_at = NOW()
    ↓
  Analyses soft-deleted too (app logic, not CASCADE)
    ↓
  After 30 days: Hard DELETE leads
    ↓
  Analyses hard-deleted (CASCADE)
    ↓
  credit_ledger.lead_id → NULL (audit preserved)
    ↓
  Transaction record: "2 credits charged for [deleted lead]"
```

---

### **11. DEDUPLICATION: Application-Level Check**

**No database constraint** - flexible business logic

```javascript
// Before creating analysis
const existing = await db.query(`
  SELECT id, status 
  FROM analyses 
  WHERE lead_id = $1 
    AND status IN ('pending', 'processing')
    AND deleted_at IS NULL
  LIMIT 1
`, [leadId]);

if (existing.rows.length > 0) {
  return { error: 'Analysis already in progress' };
}
```

**Why No DB Constraint:**
- Allows re-analysis after completion
- Supports future: scheduled re-analysis, comparison mode
- Easier to change business rules

---

### **12. PLANS AS DATA (Not Hardcoded)**

```sql
plans (
  id text PRIMARY KEY,  -- 'free', 'pro', 'enterprise'
  name text,
  credits_per_month integer,
  price_cents integer,
  stripe_price_id text,
  features jsonb
)

INSERT INTO plans VALUES
  ('free', 'Free Plan', 25, 0, NULL, '{"max_profiles": 10}'),
  ('pro', 'Pro Plan', 500, 9700, 'price_1ABC', '{"max_profiles": 1000}');
```

**Benefits:**
- Change credits without code deploy ("Black Friday: 50 free credits!")
- Add new plans without schema changes
- Feature flags per plan in JSONB

---

### **13. NAMING CONVENTIONS (Strict Standards)**

| Pattern | Examples | Rationale |
|---------|----------|-----------|
| Timestamps end in `_at` | `created_at`, `deleted_at`, `completed_at` | Consistency |
| `status` = mutable state | `'pending'`, `'active'`, `'canceled'` | Lifecycle changes |
| `{thing}_type` = category | `plan_type`, `analysis_type`, `transaction_type` | Immutable classification |
| Money in cents | `price_cents`, `cost_cents` | No decimals, no rounding errors |
| Duration in milliseconds | `processing_duration_ms` | Explicit unit |
| Booleans use `is_` | `is_verified`, `is_active`, `is_suspended` | Clear identification |
| Primary key always `id` | accounts(id), leads(id) | NOT `account_id` or `lead_id` |

---

### **14. PERFORMANCE INDEXES (Core Set)**

```sql
-- Webhook idempotency (critical)
CREATE UNIQUE INDEX idx_webhook_stripe_id ON webhook_events(stripe_event_id);

-- Dashboard queries
CREATE INDEX idx_leads_active ON leads(account_id, last_analyzed_at DESC) 
WHERE deleted_at IS NULL;

-- Latest analysis
CREATE INDEX idx_analyses_latest ON analyses(lead_id, completed_at DESC) 
WHERE deleted_at IS NULL;

-- Deduplication check
CREATE INDEX idx_analyses_in_progress ON analyses(lead_id, status) 
WHERE status IN ('pending', 'processing') AND deleted_at IS NULL;

-- Credit balance (will materialize later)
CREATE INDEX idx_credit_ledger_account ON credit_ledger(account_id, created_at DESC);

-- Active subscription
CREATE INDEX idx_subscriptions_active ON subscriptions(account_id, status) 
WHERE status = 'active';

-- Account membership (RLS helper)
CREATE INDEX idx_account_members_user ON account_members(user_id, account_id);

-- AI usage logs (90-day queries)
CREATE INDEX idx_ai_usage_account ON ai_usage_logs(account_id, created_at DESC);
```

---

### **15. ROW-LEVEL SECURITY (RLS) STRATEGY**

**Helper Function:**
```sql
CREATE FUNCTION auth.user_account_ids()
RETURNS SETOF uuid
LANGUAGE sql
STABLE
AS $$
  SELECT account_id 
  FROM account_members 
  WHERE user_id = auth.uid()
$$;
```

**Policy Pattern (Applied to All User-Facing Tables):**
```sql
-- Users see data for accounts they belong to
CREATE POLICY "account_access" ON leads
FOR ALL
USING (account_id IN (SELECT auth.user_account_ids()));

-- Service role (Cloudflare Worker) bypasses RLS
CREATE POLICY "service_role_bypass" ON leads
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

**Tables with RLS:**
- leads
- analyses
- business_profiles
- account_usage_summary
- credit_ledger (read-only for users)

**Tables WITHOUT RLS (Backend-Only):**
- webhook_events (service role only)
- platform_metrics_daily (admin only)
- ai_usage_logs (service role only)
- plans (public read)

---

### **16. DATA RETENTION & ARCHIVAL STRATEGY**

**Hot Data (Fast queries, recent):**
```
ai_usage_logs: 90 days rolling window
├── Partitioned by month (PostgreSQL native)
└── ~13.5M rows max at scale
```

**Warm Data (Aggregated, all history):**
```
account_usage_summary: Forever (monthly rollups)
platform_metrics_daily: Forever (daily rollups)
credit_ledger: Forever (audit requirement)
subscriptions: Forever (append-only history)
```

**Cold Data (Archived, rarely accessed):**
```
ai_usage_logs older than 90 days:
├── Export to S3/Glacier
└── Purge from database
```

**Cleanup Cron Jobs:**
```sql
-- Daily
DELETE FROM ai_usage_logs WHERE created_at < NOW() - INTERVAL '90 days';
DELETE FROM leads WHERE deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM analyses WHERE deleted_at < NOW() - INTERVAL '30 days';

-- Monthly (1st of month)
UPDATE account_usage_summary 
SET is_finalized = true
WHERE period_end = DATE_TRUNC('month', NOW());
```

---

## **📋 COMPLETE TABLE DEFINITIONS**

### **LAYER 1: IDENTITY & AUTH**

#### **users**
```sql
users (
  id uuid PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email text UNIQUE NOT NULL,
  full_name text,
  signature_name text,
  avatar_url text,
  onboarding_completed boolean DEFAULT false,
  is_admin boolean DEFAULT false,
  is_suspended boolean DEFAULT false,
  suspended_at timestamptz,
  suspended_reason text,
  last_seen_at timestamptz,
  created_at timestamptz DEFAULT NOW(),
  updated_at timestamptz DEFAULT NOW()
)
```

---

### **LAYER 2: BILLING FOUNDATION**

#### **accounts**
```sql
accounts (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  name text NOT NULL,
  slug text UNIQUE,
  deleted_at timestamptz,
  created_at timestamptz DEFAULT NOW(),
  updated_at timestamptz DEFAULT NOW()
)
```

#### **account_members**
```sql
account_members (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role text DEFAULT 'member',
  invited_by uuid REFERENCES users(id),
  joined_at timestamptz DEFAULT NOW(),
  created_at timestamptz DEFAULT NOW(),
  
  UNIQUE (account_id, user_id)
)
```

#### **plans**
```sql
plans (
  id text PRIMARY KEY,
  name text NOT NULL,
  credits_per_month integer NOT NULL,
  price_cents integer NOT NULL,
  stripe_price_id text,
  is_active boolean DEFAULT true,
  features jsonb,
  created_at timestamptz DEFAULT NOW()
)
```

#### **subscriptions**
```sql
subscriptions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
  plan_type text NOT NULL REFERENCES plans(id),
  price_cents integer NOT NULL,
  stripe_customer_id text,
  stripe_subscription_id text,
  stripe_price_id text,
  status text DEFAULT 'active',
  current_period_start timestamptz NOT NULL,
  current_period_end timestamptz NOT NULL,
  canceled_at timestamptz,
  created_at timestamptz DEFAULT NOW()
)
```

#### **credit_ledger**
```sql
credit_ledger (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
  amount integer NOT NULL,
  balance_after integer NOT NULL,
  transaction_type text NOT NULL,
  reference_type text,
  reference_id uuid,
  description text,
  metadata jsonb,
  created_by uuid REFERENCES users(id),
  created_at timestamptz DEFAULT NOW()
)
```

#### **account_usage_summary**
```sql
account_usage_summary (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
  period_start date NOT NULL,
  period_end date NOT NULL,
  credits_used integer DEFAULT 0,
  light_analyses_count integer DEFAULT 0,
  deep_analyses_count integer DEFAULT 0,
  total_analyses_count integer DEFAULT 0,
  total_cost_cents integer DEFAULT 0,
  is_finalized boolean DEFAULT false,
  created_at timestamptz DEFAULT NOW(),
  updated_at timestamptz DEFAULT NOW(),
  
  UNIQUE (account_id, period_start)
)
```

#### **platform_metrics_daily**
```sql
platform_metrics_daily (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  metric_date date NOT NULL UNIQUE,
  
  total_analyses_count integer DEFAULT 0,
  light_analyses_count integer DEFAULT 0,
  deep_analyses_count integer DEFAULT 0,
  active_accounts_count integer DEFAULT 0,
  new_accounts_count integer DEFAULT 0,
  
  openai_calls_count integer DEFAULT 0,
  anthropic_calls_count integer DEFAULT 0,
  apify_calls_count integer DEFAULT 0,
  openai_cost_cents integer DEFAULT 0,
  anthropic_cost_cents integer DEFAULT 0,
  apify_cost_cents integer DEFAULT 0,
  total_cost_cents integer DEFAULT 0,
  
  openai_avg_duration_ms integer DEFAULT 0,
  anthropic_avg_duration_ms integer DEFAULT 0,
  apify_avg_duration_ms integer DEFAULT 0,
  openai_error_count integer DEFAULT 0,
  anthropic_error_count integer DEFAULT 0,
  apify_error_count integer DEFAULT 0,
  
  credits_purchased_count integer DEFAULT 0,
  revenue_cents integer DEFAULT 0,
  profit_margin_cents integer DEFAULT 0,
  
  daily_active_users integer DEFAULT 0,
  avg_lead_score numeric,
  high_quality_leads_count integer DEFAULT 0,
  
  mrr_cents integer DEFAULT 0,
  arr_cents integer DEFAULT 0,
  customer_count integer DEFAULT 0,
  
  created_at timestamptz DEFAULT NOW(),
  updated_at timestamptz DEFAULT NOW()
)
```

#### **stripe_invoices**
```sql
stripe_invoices (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
  stripe_invoice_id text UNIQUE NOT NULL,
  stripe_subscription_id text,
  amount_cents integer NOT NULL,
  currency text DEFAULT 'usd',
  status text NOT NULL,
  paid_at timestamptz,
  failed_at timestamptz,
  failure_reason text,
  created_at timestamptz DEFAULT NOW()
)
```

#### **webhook_events**
```sql
webhook_events (
  stripe_event_id text PRIMARY KEY,
  event_type text NOT NULL,
  account_id uuid REFERENCES accounts(id),
  processed_at timestamptz DEFAULT NOW(),
  payload jsonb
)
```

---

### **LAYER 3: BUSINESS LOGIC**

#### **business_profiles**
```sql
business_profiles (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
  
  business_name text NOT NULL,
  website text,
  business_one_liner text,
  
  business_context_pack jsonb,
  
  context_version text DEFAULT 'v1.0',
  context_generated_at timestamptz,
  context_manually_edited boolean DEFAULT false,
  context_updated_at timestamptz,
  
  is_active boolean DEFAULT true,
  deleted_at timestamptz,
  created_at timestamptz DEFAULT NOW(),
  updated_at timestamptz DEFAULT NOW()
)
```

#### **leads**
```sql
leads (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
  business_profile_id uuid REFERENCES business_profiles(id) ON DELETE SET NULL,
  
  instagram_username text UNIQUE NOT NULL,
  display_name text,
  profile_pic_url text,
  profile_url text,
  follower_count integer,
  following_count integer,
  post_count integer,
  bio text,
  external_url text,
  is_verified boolean DEFAULT false,
  is_private boolean DEFAULT false,
  is_business_account boolean DEFAULT false,
  platform text DEFAULT 'instagram',
  
  first_analyzed_at timestamptz,
  last_analyzed_at timestamptz,
  deleted_at timestamptz,
  created_at timestamptz DEFAULT NOW()
)
```

#### **analyses**
```sql
analyses (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id uuid NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
  account_id uuid NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
  business_profile_id uuid REFERENCES business_profiles(id) ON DELETE SET NULL,
  
  analysis_type text NOT NULL,
  analysis_version text,
  status text DEFAULT 'pending',
  
  ai_response jsonb,
  
  overall_score integer,
  niche_fit_score integer,
  engagement_score integer,
  confidence_level numeric,
  
  credits_charged integer,
  total_cost_cents integer,
  model_used text,
  processing_duration_ms integer,
  
  started_at timestamptz,
  completed_at timestamptz,
  deleted_at timestamptz,
  created_at timestamptz DEFAULT NOW()
)
```

#### **ai_usage_logs**
```sql
ai_usage_logs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
  analysis_id uuid REFERENCES analyses(id) ON DELETE SET NULL,
  
  provider text NOT NULL,
  model text,
  api_call_type text,
  
  tokens_input integer,
  tokens_output integer,
  cost_cents integer NOT NULL,
  
  duration_ms integer,
  status text,
  error_message text,
  
  started_at timestamptz,
  completed_at timestamptz,
  created_at timestamptz DEFAULT NOW()
)
```

---

## **🔧 PHASE 1: PRE-LAUNCH CRITICAL (1-2 Days)**

### **1. Foreign Key Constraints**
```sql
-- All relationships explicitly defined with cascade behavior
ALTER TABLE account_members ADD CONSTRAINT fk_account 
  FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE CASCADE;

ALTER TABLE account_members ADD CONSTRAINT fk_user 
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE;

ALTER TABLE subscriptions ADD CONSTRAINT fk_account 
  FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE RESTRICT;

ALTER TABLE credit_ledger ADD CONSTRAINT fk_account 
  FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE RESTRICT;

ALTER TABLE leads ADD CONSTRAINT fk_account 
  FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE RESTRICT;

ALTER TABLE leads ADD CONSTRAINT fk_business_profile 
  FOREIGN KEY (business_profile_id) REFERENCES business_profiles(id) ON DELETE SET NULL;

ALTER TABLE analyses ADD CONSTRAINT fk_lead 
  FOREIGN KEY (lead_id) REFERENCES leads(id) ON DELETE CASCADE;

ALTER TABLE analyses ADD CONSTRAINT fk_account 
  FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE RESTRICT;

ALTER TABLE analyses ADD CONSTRAINT fk_business_profile 
  FOREIGN KEY (business_profile_id) REFERENCES business_profiles(id) ON DELETE SET NULL;

ALTER TABLE ai_usage_logs ADD CONSTRAINT fk_account 
  FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE RESTRICT;

ALTER TABLE ai_usage_logs ADD CONSTRAINT fk_analysis 
  FOREIGN KEY (analysis_id) REFERENCES analyses(id) ON DELETE SET NULL;

-- Continue for all FK relationships...
```

---

### **2. Unique Constraints**
```sql
-- One active subscription per account
CREATE UNIQUE INDEX idx_subscriptions_active_per_account 
ON subscriptions(account_id) 
WHERE status = 'active';

-- No duplicate account members
-- (Already in table definition: UNIQUE (account_id, user_id))

-- No duplicate Instagram usernames
-- (Already in table definition: instagram_username text UNIQUE)

-- One usage summary per account per period
-- (Already in table definition: UNIQUE (account_id, period_start))

-- Unique account slugs
-- (Already in table definition: slug text UNIQUE)

-- Unique Stripe event IDs
-- (Already in table definition: stripe_event_id text PRIMARY KEY)

-- Unique Stripe invoice IDs
-- (Already in table definition: stripe_invoice_id text UNIQUE)
```

---

### **3. Check Constraints**
```sql
-- Credit balance cannot be negative
ALTER TABLE credit_ledger
ADD CONSTRAINT check_balance_non_negative
CHECK (balance_after >= 0);

-- AI costs cannot be negative
ALTER TABLE ai_usage_logs
ADD CONSTRAINT check_cost_non_negative
CHECK (cost_cents >= 0);

-- Valid subscription statuses
ALTER TABLE subscriptions
ADD CONSTRAINT check_valid_status
CHECK (status IN ('active', 'canceled', 'past_due', 'paused', 'pending_stripe_confirmation'));

-- Valid analysis statuses
ALTER TABLE analyses
ADD CONSTRAINT check_valid_status
CHECK (status IN ('pending', 'processing', 'completed', 'failed'));

-- Valid account member roles
ALTER TABLE account_members
ADD CONSTRAINT check_valid_role
CHECK (role IN ('owner', 'admin', 'member'));

-- Valid transaction types
ALTER TABLE credit_ledger
ADD CONSTRAINT check_valid_transaction_type
CHECK (transaction_type IN ('subscription_renewal', 'analysis', 'refund', 'admin_grant', 'chargeback'));

-- Valid analysis types
ALTER TABLE analyses
ADD CONSTRAINT check_valid_analysis_type
CHECK (analysis_type IN ('light', 'deep'));

-- Valid AI providers
ALTER TABLE ai_usage_logs
ADD CONSTRAINT check_valid_provider
CHECK (provider IN ('openai', 'anthropic', 'apify'));

-- Valid AI call statuses
ALTER TABLE ai_usage_logs
ADD CONSTRAINT check_valid_ai_status
CHECK (status IN ('success', 'error', 'timeout'));

-- Prices must be non-negative
ALTER TABLE subscriptions
ADD CONSTRAINT check_price_non_negative
CHECK (price_cents >= 0);

ALTER TABLE plans
ADD CONSTRAINT check_price_non_negative
CHECK (price_cents >= 0);

ALTER TABLE plans
ADD CONSTRAINT check_credits_non_negative
CHECK (credits_per_month >= 0);
```

---

### **4. Triggers (Auto-Update Timestamps)**
```sql
-- Generic trigger function
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to tables with updated_at column
CREATE TRIGGER update_users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_accounts_updated_at
  BEFORE UPDATE ON accounts
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_business_profiles_updated_at
  BEFORE UPDATE ON business_profiles
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_account_usage_summary_updated_at
  BEFORE UPDATE ON account_usage_summary
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_platform_metrics_updated_at
  BEFORE UPDATE ON platform_metrics_daily
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

---

### **5. Row-Level Security (RLS) Policies**
```sql
-- Helper function
CREATE FUNCTION auth.user_account_ids()
RETURNS SETOF uuid
LANGUAGE sql
STABLE
AS $$
  SELECT account_id 
  FROM account_members 
  WHERE user_id = auth.uid()
$$;

-- Enable RLS on user-facing tables
ALTER TABLE leads ENABLE ROW LEVEL SECURITY;
ALTER TABLE analyses ENABLE ROW LEVEL SECURITY;
ALTER TABLE business_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE account_usage_summary ENABLE ROW LEVEL SECURITY;
ALTER TABLE credit_ledger ENABLE ROW LEVEL SECURITY;

-- Policies: Account-based access
CREATE POLICY "account_access" ON leads
FOR ALL
USING (account_id IN (SELECT auth.user_account_ids()));

CREATE POLICY "account_access" ON analyses
FOR ALL
USING (account_id IN (SELECT auth.user_account_ids()));

CREATE POLICY "account_access" ON business_profiles
FOR ALL
USING (account_id IN (SELECT auth.user_account_ids()));

CREATE POLICY "account_access" ON account_usage_summary
FOR ALL
USING (account_id IN (SELECT auth.user_account_ids()));

CREATE POLICY "account_access_read_only" ON credit_ledger
FOR SELECT
USING (account_id IN (SELECT auth.user_account_ids()));

-- Service role bypass (Cloudflare Worker)
CREATE POLICY "service_role_bypass" ON leads
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');

CREATE POLICY "service_role_bypass" ON analyses
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');

CREATE POLICY "service_role_bypass" ON business_profiles
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');

CREATE POLICY "service_role_bypass" ON account_usage_summary
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');

CREATE POLICY "service_role_bypass" ON credit_ledger
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

---

### **6. Core Indexes**
```sql
-- (Already listed in decision #15 above)
CREATE UNIQUE INDEX idx_webhook_stripe_id ON webhook_events(stripe_event_id);
CREATE INDEX idx_leads_active ON leads(account_id, last_analyzed_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX idx_analyses_latest ON analyses(lead_id, completed_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX idx_analyses_in_progress ON analyses(lead_id, status) WHERE status IN ('pending', 'processing') AND deleted_at IS NULL;
CREATE INDEX idx_credit_ledger_account ON credit_ledger(account_id, created_at DESC);
CREATE INDEX idx_subscriptions_active ON subscriptions(account_id, status) WHERE status = 'active';
CREATE INDEX idx_account_members_user ON account_members(user_id, account_id);
CREATE INDEX idx_ai_usage_account ON ai_usage_logs(account_id, created_at DESC);
CREATE INDEX idx_platform_metrics_date ON platform_metrics_daily(metric_date DESC);
CREATE INDEX idx_account_usage ON account_usage_summary(account_id, period_start DESC);
```

---

## **🔧 PHASE 2: FIRST WEEK LIVE (Performance Optimization)**

### **7. Materialized Credit Balance Table**
```sql
CREATE TABLE credit_balances (
  account_id uuid PRIMARY KEY REFERENCES accounts(id) ON DELETE CASCADE,
  current_balance integer NOT NULL,
  last_transaction_at timestamptz,
  updated_at timestamptz DEFAULT NOW()
);

-- Trigger to update on every credit_ledger INSERT
CREATE FUNCTION update_credit_balance()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO credit_balances (account_id, current_balance, last_transaction_at)
  VALUES (NEW.account_id, NEW.balance_after, NOW())
  ON CONFLICT (account_id)
  DO UPDATE SET
    current_balance = NEW.balance_after,
    last_transaction_at = NOW(),
    updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER credit_ledger_update_balance
  AFTER INSERT ON credit_ledger
  FOR EACH ROW
  EXECUTE FUNCTION update_credit_balance();

-- Backfill existing balances
INSERT INTO credit_balances (account_id, current_balance, last_transaction_at)
SELECT 
  account_id,
  SUM(amount) as current_balance,
  MAX(created_at) as last_transaction_at
FROM credit_ledger
GROUP BY account_id
ON CONFLICT (account_id) DO NOTHING;
```

**Query Pattern (Fast):**
```sql
-- Old way (slow at scale)
SELECT SUM(amount) FROM credit_ledger WHERE account_id = 'acc_123';

-- New way (instant)
SELECT current_balance FROM credit_balances WHERE account_id = 'acc_123';
```

---

### **8. Daily Cleanup Cron Job**
```javascript
// Cloudflare Worker scheduled job (runs daily at 2 AM UTC)
export async function scheduledCleanup() {
  
  // 1. Archive old ai_usage_logs (older than 90 days)
  await db.query(`
    DELETE FROM ai_usage_logs 
    WHERE created_at < NOW() - INTERVAL '90 days'
  `);
  
  // 2. Hard-delete soft-deleted records (older than 30 days)
  await db.query(`
    DELETE FROM leads 
    WHERE deleted_at < NOW() - INTERVAL '30 days'
  `);
  
  await db.query(`
    DELETE FROM analyses 
    WHERE deleted_at < NOW() - INTERVAL '30 days'
  `);
  
  await db.query(`
    DELETE FROM business_profiles 
    WHERE deleted_at < NOW() - INTERVAL '30 days'
  `);
  
  await db.query(`
    DELETE FROM accounts 
    WHERE deleted_at < NOW() - INTERVAL '30 days'
  `);
  
  console.log('Daily cleanup completed');
}
```

---

### **9. Monthly Rollup Cron Job**
```javascript
// Cloudflare Worker scheduled job (runs 1st of month at 3 AM UTC)
export async function monthlyRollup() {
  
  // 1. Freeze last month's usage summaries
  await db.query(`
    UPDATE account_usage_summary
    SET is_finalized = true
    WHERE period_end = DATE_TRUNC('month', NOW())
      AND is_finalized = false
  `);
  
  // 2. Grant free plan monthly credits
  const freeAccounts = await db.query(`
    SELECT a.id as account_id, p.credits_per_month
    FROM accounts a
    JOIN subscriptions s ON s.account_id = a.id
    JOIN plans p ON p.id = s.plan_type
    WHERE s.status = 'active'
      AND p.id = 'free'
      AND a.deleted_at IS NULL
  `);
  
  for (const account of freeAccounts.rows) {
    const currentBalance = await db.query(`
      SELECT current_balance 
      FROM credit_balances 
      WHERE account_id = $1
    `, [account.account_id]);
    
    const newBalance = currentBalance.rows[0].current_balance + account.credits_per_month;
    
    await db.query(`
      INSERT INTO credit_ledger (
        account_id, 
        amount, 
        balance_after, 
        transaction_type,
        description
      ) VALUES ($1, $2, $3, 'subscription_renewal', 'Monthly free plan credit grant')
    `, [account.account_id, account.credits_per_month, newBalance]);
  }
  
  console.log('Monthly rollup completed');
}
```

---

## **📝 CRITICAL IMPLEMENTATION NOTES**

### **Data Type Warnings**
```sql
-- ❌ WRONG (will break Stripe integration)
stripe_customer_id uuid
stripe_subscription_id uuid

-- ✅ CORRECT
stripe_customer_id text  -- Stripe returns "cus_abc123..."
stripe_subscription_id text  -- Stripe returns "sub_xyz789..."
stripe_price_id text  -- Stripe returns "price_1ABC..."
stripe_event_id text  -- Stripe returns "evt_1A2B3C..."
stripe_invoice_id text  -- Stripe returns "in_1DEF456..."
```

### **Query Patterns**
```sql
-- Always filter deleted records
SELECT * FROM leads WHERE deleted_at IS NULL;

-- Get latest analysis for lead
SELECT * FROM analyses 
WHERE lead_id = $1 AND deleted_at IS NULL
ORDER BY completed_at DESC 
LIMIT 1;

-- Check for in-progress analysis (deduplication)
SELECT * FROM analyses
WHERE lead_id = $1 
  AND status IN ('pending', 'processing')
  AND deleted_at IS NULL
LIMIT 1;

-- Get credit balance (after materialization)
SELECT current_balance FROM credit_balances WHERE account_id = $1;

-- Get active subscription
SELECT * FROM subscriptions 
WHERE account_id = $1 AND status = 'active'
LIMIT 1;
```

---

## **✅ SCHEMA STATUS**

**14 Tables Total:**
- Layer 1 (Identity): 1 table
- Layer 2 (Billing): 9 tables (includes credit_balances from Phase 2)
- Layer 3 (Business Logic): 4 tables

**All Architectural Decisions:** ✅ Locked  
**Phase 1 Requirements:** ✅ Defined  
**Phase 2 Requirements:** ✅ Defined  
**Phase 3:** ⏸️ Deferred

**READY FOR SQL GENERATION** 

---

*This document contains the complete specification for Oslira Database V2. Provide this to any AI assistant for implementation guidance.*

# 🔒 PHASE 1: COMPLETE RLS IMPLEMENTATION

**Add this section after Phase 0 in your database document**

---

## **📋 RLS STRATEGY OVERVIEW**

### **Security Model:**
- ✅ All user-facing tables have RLS enabled
- ✅ Users only access their own account data
- ✅ Soft-deleted records hidden by default
- ✅ Service role bypasses all policies (Cloudflare Worker)
- ✅ Admin users bypass all policies (support/debugging)
- ✅ No anonymous access (authentication required)

### **Tables with RLS:**
- accounts (strict - only user's accounts)
- account_members (owners manage members)
- leads (account-scoped, active only)
- analyses (account-scoped, active only)
- business_profiles (account-scoped, active only)
- credit_ledger (read-only for users)
- account_usage_summary (read-only for users)
- subscriptions (read-only for users)
- stripe_invoices (read-only for users)

### **Tables WITHOUT RLS (Service Role Only):**
- webhook_events
- ai_usage_logs
- platform_metrics_daily
- plans (public read access)

---

## **🛠️ HELPER FUNCTIONS**

### **FUNCTION #1: Get User's Account IDs**

```sql
CREATE OR REPLACE FUNCTION auth.user_account_ids()
RETURNS SETOF uuid
LANGUAGE sql
STABLE
SECURITY DEFINER
AS $$
  SELECT am.account_id 
  FROM account_members am
  JOIN accounts a ON a.id = am.account_id
  WHERE am.user_id = auth.uid()
    AND a.deleted_at IS NULL
    AND a.is_suspended = false
$$;
```

**What it does:**
- Returns all account IDs the current user belongs to
- Excludes deleted accounts
- Excludes suspended accounts
- Used in every RLS policy for account-scoped data

**Usage in policies:**
```sql
WHERE account_id IN (SELECT auth.user_account_ids())
```

---

### **FUNCTION #2: Check if User is Admin**

```sql
CREATE OR REPLACE FUNCTION auth.is_admin()
RETURNS boolean
LANGUAGE sql
STABLE
SECURITY DEFINER
AS $$
  SELECT EXISTS (
    SELECT 1 FROM users 
    WHERE id = auth.uid() 
    AND is_admin = true
    AND is_suspended = false
  )
$$;
```

**What it does:**
- Returns true if current user has admin flag
- Excludes suspended admins
- Used in all policies for admin bypass

**Usage in policies:**
```sql
WHERE auth.is_admin() OR (account_id IN ...)
```

---

## **🔐 CORE RLS POLICIES**

### **1. ACCOUNTS TABLE**

```sql
-- Enable RLS
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

-- Policy: Users see only their own accounts
CREATE POLICY "users_see_own_accounts" ON accounts
FOR SELECT
USING (
  auth.is_admin()
  OR id IN (SELECT auth.user_account_ids())
);

-- Policy: Users can update their own accounts
CREATE POLICY "users_update_own_accounts" ON accounts
FOR UPDATE
USING (
  auth.is_admin()
  OR (id IN (SELECT auth.user_account_ids()) AND deleted_at IS NULL)
);

-- Policy: Users can soft-delete their own accounts (owners only)
CREATE POLICY "owners_delete_accounts" ON accounts
FOR UPDATE
USING (
  auth.is_admin()
  OR (owner_id = auth.uid() AND deleted_at IS NULL)
);

-- Policy: Service role bypass
CREATE POLICY "service_role_full_access" ON accounts
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

**Note:** Users CANNOT INSERT accounts directly (signup flow via service role)

---

### **2. ACCOUNT_MEMBERS TABLE**

```sql
-- Enable RLS
ALTER TABLE account_members ENABLE ROW LEVEL SECURITY;

-- Policy: Users see members of their accounts
CREATE POLICY "users_see_members" ON account_members
FOR SELECT
USING (
  auth.is_admin()
  OR account_id IN (SELECT auth.user_account_ids())
);

-- Policy: Account owners manage members (Phase 2 - currently disabled)
-- Uncomment when member invite feature is ready
/*
CREATE POLICY "owners_manage_members" ON account_members
FOR ALL
USING (
  auth.is_admin()
  OR account_id IN (
    SELECT id FROM accounts 
    WHERE owner_id = auth.uid() 
    AND deleted_at IS NULL
  )
);
*/

-- Policy: Service role bypass
CREATE POLICY "service_role_full_access" ON account_members
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

**Phase 2 Note:** Uncomment `owners_manage_members` policy when adding team invite feature

---

### **3. LEADS TABLE**

```sql
-- Enable RLS
ALTER TABLE leads ENABLE ROW LEVEL SECURITY;

-- Policy: Users access leads from their accounts (active only)
CREATE POLICY "account_access_active" ON leads
FOR SELECT
USING (
  auth.is_admin()
  OR (
    account_id IN (SELECT auth.user_account_ids())
    AND deleted_at IS NULL
  )
);

-- Policy: Users can create leads in their accounts
CREATE POLICY "users_create_leads" ON leads
FOR INSERT
WITH CHECK (
  auth.is_admin()
  OR account_id IN (SELECT auth.user_account_ids())
);

-- Policy: Users can update leads in their accounts
CREATE POLICY "users_update_leads" ON leads
FOR UPDATE
USING (
  auth.is_admin()
  OR (
    account_id IN (SELECT auth.user_account_ids())
    AND deleted_at IS NULL
  )
);

-- Policy: Users can soft-delete leads (sets deleted_at)
CREATE POLICY "users_delete_leads" ON leads
FOR UPDATE
USING (
  auth.is_admin()
  OR (
    account_id IN (SELECT auth.user_account_ids())
    AND deleted_at IS NULL
  )
);

-- OPTIONAL: Trash access (view deleted leads within 30 days)
-- Uncomment if you want "Trash" feature
/*
CREATE POLICY "trash_access" ON leads
FOR SELECT
USING (
  auth.is_admin()
  OR (
    account_id IN (SELECT auth.user_account_ids())
    AND deleted_at IS NOT NULL
    AND deleted_at > NOW() - INTERVAL '30 days'
  )
);
*/

-- Policy: Service role bypass
CREATE POLICY "service_role_full_access" ON leads
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

---

### **4. ANALYSES TABLE**

```sql
-- Enable RLS
ALTER TABLE analyses ENABLE ROW LEVEL SECURITY;

-- Policy: Users access analyses from their accounts (active only)
CREATE POLICY "account_access_active" ON analyses
FOR SELECT
USING (
  auth.is_admin()
  OR (
    account_id IN (SELECT auth.user_account_ids())
    AND deleted_at IS NULL
  )
);

-- Policy: Users can create analyses (via service role during analysis flow)
CREATE POLICY "users_create_analyses" ON analyses
FOR INSERT
WITH CHECK (
  auth.is_admin()
  OR account_id IN (SELECT auth.user_account_ids())
);

-- Policy: Users can update analyses
CREATE POLICY "users_update_analyses" ON analyses
FOR UPDATE
USING (
  auth.is_admin()
  OR (
    account_id IN (SELECT auth.user_account_ids())
    AND deleted_at IS NULL
  )
);

-- OPTIONAL: Trash access
/*
CREATE POLICY "trash_access" ON analyses
FOR SELECT
USING (
  auth.is_admin()
  OR (
    account_id IN (SELECT auth.user_account_ids())
    AND deleted_at IS NOT NULL
    AND deleted_at > NOW() - INTERVAL '30 days'
  )
);
*/

-- Policy: Service role bypass
CREATE POLICY "service_role_full_access" ON analyses
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

---

### **5. BUSINESS_PROFILES TABLE**

```sql
-- Enable RLS
ALTER TABLE business_profiles ENABLE ROW LEVEL SECURITY;

-- Policy: Users access business profiles from their accounts (active only)
CREATE POLICY "account_access_active" ON business_profiles
FOR SELECT
USING (
  auth.is_admin()
  OR (
    account_id IN (SELECT auth.user_account_ids())
    AND deleted_at IS NULL
  )
);

-- Policy: Users can create business profiles
CREATE POLICY "users_create_profiles" ON business_profiles
FOR INSERT
WITH CHECK (
  auth.is_admin()
  OR account_id IN (SELECT auth.user_account_ids())
);

-- Policy: Users can update business profiles
CREATE POLICY "users_update_profiles" ON business_profiles
FOR UPDATE
USING (
  auth.is_admin()
  OR (
    account_id IN (SELECT auth.user_account_ids())
    AND deleted_at IS NULL
  )
);

-- OPTIONAL: Trash access
/*
CREATE POLICY "trash_access" ON business_profiles
FOR SELECT
USING (
  auth.is_admin()
  OR (
    account_id IN (SELECT auth.user_account_ids())
    AND deleted_at IS NOT NULL
    AND deleted_at > NOW() - INTERVAL '30 days'
  )
);
*/

-- Policy: Service role bypass
CREATE POLICY "service_role_full_access" ON business_profiles
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

---

### **6. CREDIT_LEDGER TABLE (Read-Only)**

```sql
-- Enable RLS
ALTER TABLE credit_ledger ENABLE ROW LEVEL SECURITY;

-- Policy: Users can READ their credit history
CREATE POLICY "users_read_credit_history" ON credit_ledger
FOR SELECT
USING (
  auth.is_admin()
  OR account_id IN (SELECT auth.user_account_ids())
);

-- Policy: Service role bypass (INSERT/UPDATE only via service role)
CREATE POLICY "service_role_full_access" ON credit_ledger
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

**Note:** Users can only SELECT, never INSERT/UPDATE/DELETE (managed by `deduct_credits()` function)

---

### **7. ACCOUNT_USAGE_SUMMARY TABLE (Read-Only)**

```sql
-- Enable RLS
ALTER TABLE account_usage_summary ENABLE ROW LEVEL SECURITY;

-- Policy: Users can READ their usage stats
CREATE POLICY "users_read_usage" ON account_usage_summary
FOR SELECT
USING (
  auth.is_admin()
  OR account_id IN (SELECT auth.user_account_ids())
);

-- Policy: Service role bypass
CREATE POLICY "service_role_full_access" ON account_usage_summary
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

---

### **8. SUBSCRIPTIONS TABLE (Read-Only)**

```sql
-- Enable RLS
ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;

-- Policy: Users can READ their subscription details
CREATE POLICY "users_read_subscription" ON subscriptions
FOR SELECT
USING (
  auth.is_admin()
  OR account_id IN (SELECT auth.user_account_ids())
);

-- Policy: Service role bypass
CREATE POLICY "service_role_full_access" ON subscriptions
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

---

### **9. STRIPE_INVOICES TABLE (Read-Only)**

```sql
-- Enable RLS
ALTER TABLE stripe_invoices ENABLE ROW LEVEL SECURITY;

-- Policy: Users can READ their billing history
CREATE POLICY "users_read_invoices" ON stripe_invoices
FOR SELECT
USING (
  auth.is_admin()
  OR account_id IN (SELECT auth.user_account_ids())
);

-- Policy: Service role bypass
CREATE POLICY "service_role_full_access" ON stripe_invoices
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

---

## **🌐 PUBLIC ACCESS TABLES**

### **PLANS TABLE (Public Read)**

```sql
-- Enable RLS
ALTER TABLE plans ENABLE ROW LEVEL SECURITY;

-- Policy: Anyone can read plans (for pricing page)
CREATE POLICY "public_read_plans" ON plans
FOR SELECT
USING (true);

-- Policy: Service role can manage plans
CREATE POLICY "service_role_full_access" ON plans
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

---

## **🚫 SERVICE ROLE ONLY TABLES**

These tables have **NO RLS** - only accessible via service role (Cloudflare Worker)

### **Tables:**
- `webhook_events` - Stripe webhook processing
- `ai_usage_logs` - API cost tracking
- `platform_metrics_daily` - Admin analytics
- `credit_balances` - Materialized view (managed by triggers)

**No policies needed** - service role has full access by default

---

## **✅ RLS TESTING CHECKLIST**

Run these tests in Supabase SQL Editor to verify policies work:

### **Test 1: User Cannot See Other Accounts**
```sql
-- Login as user A
SET request.jwt.claims.sub = 'user_a_uuid';

-- Try to query all accounts
SELECT * FROM accounts;
-- Should return ONLY user A's accounts
```

### **Test 2: User Cannot Access Other Account's Leads**
```sql
-- Login as user A
SET request.jwt.claims.sub = 'user_a_uuid';

-- Try to query leads from account B
SELECT * FROM leads WHERE account_id = 'account_b_uuid';
-- Should return EMPTY (access denied)
```

### **Test 3: User Cannot Insert Credits Directly**
```sql
-- Login as user A
SET request.jwt.claims.sub = 'user_a_uuid';

-- Try to insert credit transaction
INSERT INTO credit_ledger (account_id, amount, balance_after, ...)
VALUES ('user_a_account_id', 1000, 1000, ...);
-- Should FAIL (only service role can INSERT)
```

### **Test 4: User Can Read Their Credit History**
```sql
-- Login as user A
SET request.jwt.claims.sub = 'user_a_uuid';

-- Read credit history
SELECT * FROM credit_ledger WHERE account_id IN (SELECT auth.user_account_ids());
-- Should return user A's transactions
```

### **Test 5: Admin Sees Everything**
```sql
-- Set user as admin
UPDATE users SET is_admin = true WHERE id = 'user_a_uuid';

-- Login as admin
SET request.jwt.claims.sub = 'user_a_uuid';

-- Query all leads
SELECT COUNT(*) FROM leads;
-- Should return ALL leads from ALL accounts
```

### **Test 6: Soft-Deleted Records Hidden**
```sql
-- Soft delete a lead
UPDATE leads SET deleted_at = NOW() WHERE id = 'lead_123';

-- Login as owner of that lead
SET request.jwt.claims.sub = 'owner_uuid';

-- Try to query
SELECT * FROM leads WHERE id = 'lead_123';
-- Should return EMPTY (deleted_at IS NOT NULL filtered by RLS)
```

### **Test 7: Service Role Bypasses All**
```sql
-- Use service role key (in Cloudflare Worker)
-- Query with SUPABASE_SERVICE_ROLE

SELECT * FROM leads;
-- Should return ALL leads (including deleted, suspended accounts)
```

---

## **🔧 GRANTING ADMIN ACCESS**

### **Manual SQL (Temporary)**
```sql
-- Grant admin to specific user
UPDATE users 
SET is_admin = true 
WHERE email = 'your-email@example.com';

-- Revoke admin
UPDATE users 
SET is_admin = false 
WHERE id = 'user_uuid';
```

### **Via Service Role API (Production)**
```javascript
// Admin management endpoint
// POST /api/admin/grant-access
export async function grantAdmin(userId) {
  // Verify caller is existing admin
  const caller = await getCurrentUser();
  if (!caller.is_admin) {
    throw new Error('Unauthorized');
  }
  
  // Grant admin access
  await db.query(`
    UPDATE users 
    SET is_admin = true,
        updated_at = NOW()
    WHERE id = $1
  `, [userId]);
  
  // Log action
  console.log(`Admin granted to ${userId} by ${caller.id}`);
}
```

---

## **📊 RLS PERFORMANCE NOTES**

### **Indexes Required for Fast RLS:**
```sql
-- Account membership lookups (used in EVERY policy)
CREATE INDEX idx_account_members_user 
ON account_members(user_id, account_id);

-- Account soft delete checks
CREATE INDEX idx_accounts_active 
ON accounts(id) 
WHERE deleted_at IS NULL AND is_suspended = false;

-- Leads queries
CREATE INDEX idx_leads_account_active 
ON leads(account_id, created_at DESC) 
WHERE deleted_at IS NULL;

-- Analyses queries
CREATE INDEX idx_analyses_account_active 
ON analyses(account_id, created_at DESC) 
WHERE deleted_at IS NULL;
```

These indexes are already defined in your main document - ensure they're deployed.

---

## **🚨 COMMON RLS PITFALLS TO AVOID**

### **Pitfall #1: Forgetting Service Role Bypass**
```sql
-- ❌ WRONG: No service role policy
CREATE POLICY "user_access" ON leads ...;
-- Worker calls fail!

-- ✅ CORRECT: Always add service role bypass
CREATE POLICY "service_role_full_access" ON leads
FOR ALL USING (auth.jwt() ->> 'role' = 'service_role');
```

### **Pitfall #2: RLS on Junction Tables**
```sql
-- account_members needs RLS too!
-- Without it, users can add themselves to any account
ALTER TABLE account_members ENABLE ROW LEVEL SECURITY;
```

### **Pitfall #3: Suspended Account Access**
```sql
-- ❌ WRONG: Doesn't check suspension
WHERE account_id IN (SELECT account_id FROM account_members WHERE user_id = auth.uid())

-- ✅ CORRECT: Use helper function (includes suspension check)
WHERE account_id IN (SELECT auth.user_account_ids())
```

### **Pitfall #4: Missing deleted_at Filter**
```sql
-- ❌ WRONG: Shows deleted records
WHERE account_id IN (SELECT auth.user_account_ids())

-- ✅ CORRECT: Filter deleted
WHERE account_id IN (SELECT auth.user_account_ids())
  AND deleted_at IS NULL
```

---

## **📋 DEPLOYMENT CHECKLIST**

- [ ] Created helper functions (`auth.user_account_ids()`, `auth.is_admin()`)
- [ ] Enabled RLS on all user-facing tables
- [ ] Created policies for: accounts, account_members, leads, analyses, business_profiles
- [ ] Created read-only policies for: credit_ledger, account_usage_summary, subscriptions, stripe_invoices
- [ ] Created public read policy for: plans
- [ ] Verified service role bypass on all tables
- [ ] Added required indexes for RLS performance
- [ ] Tested with non-admin user (cannot see other accounts)
- [ ] Tested with admin user (sees everything)
- [ ] Tested service role access (bypasses all RLS)
- [ ] Verified soft-deleted records are hidden
- [ ] Verified suspended accounts are inaccessible

---

## **🎯 NEXT STEPS**

### **After RLS Deployment:**
1. Test thoroughly with real user accounts
2. Monitor query performance (should be <50ms for most queries)
3. Add trash/restore feature (uncomment optional policies)
4. Implement team invite feature (uncomment `owners_manage_members` policy)

### **Phase 2 Features:**
- Role-based permissions (admin vs member)
- Trash/restore functionality
- Team member invitations
- Audit logging for sensitive operations

---

**STATUS: RLS Implementation Complete ✅**

All tables are secured with row-level security policies. Users can only access their own account data, admins bypass all restrictions, and service role has full access for backend operations.

# 🔧 MISSING DOCUMENTATION SECTIONS

**Add these sections to your existing database v2 document**

---

## **📍 ADD TO PHASE 0: Credit Balance Trigger**

**Insert this after the `deduct_credits()` function:**

### **Credit Balance Materialization Trigger**

**Purpose:** Auto-update `credit_balances` table whenever `credit_ledger` changes (eliminates need for SUM() queries)

```sql
-- Trigger function
CREATE OR REPLACE FUNCTION update_credit_balance()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO credit_balances (account_id, current_balance, last_transaction_at)
  VALUES (NEW.account_id, NEW.balance_after, NOW())
  ON CONFLICT (account_id)
  DO UPDATE SET
    current_balance = NEW.balance_after,
    last_transaction_at = NOW(),
    updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Attach trigger to credit_ledger
CREATE TRIGGER credit_ledger_update_balance
AFTER INSERT ON credit_ledger
FOR EACH ROW
EXECUTE FUNCTION update_credit_balance();
```

**How it works:**
1. `deduct_credits()` inserts into `credit_ledger`
2. Trigger fires automatically
3. `credit_balances.current_balance` updated instantly
4. Dashboard queries `credit_balances` (fast) instead of `SUM(credit_ledger)` (slow)

**Backfill existing balances:**
```sql
INSERT INTO credit_balances (account_id, current_balance, last_transaction_at)
SELECT 
  account_id,
  SUM(amount) as current_balance,
  MAX(created_at) as last_transaction_at
FROM credit_ledger
GROUP BY account_id
ON CONFLICT (account_id) DO NOTHING;
```

---

## **📍 ADD TO PHASE 1: Additional Constraint**

**Insert this in the "Unique Constraints" section:**

### **One Active Subscription Per Account**

```sql
-- Prevent duplicate active subscriptions
CREATE UNIQUE INDEX idx_one_active_subscription
ON subscriptions(account_id)
WHERE status = 'active';
```

**Why critical:** Without this, bugs could create multiple active subscriptions:
```sql
-- ❌ WITHOUT CONSTRAINT (possible bug):
INSERT INTO subscriptions (account_id, plan_type, status) 
VALUES ('acc_123', 'free', 'active');

INSERT INTO subscriptions (account_id, plan_type, status) 
VALUES ('acc_123', 'pro', 'active');
-- Both succeed! Which is the real subscription?

-- ✅ WITH CONSTRAINT:
-- Second INSERT fails with: "duplicate key value violates unique constraint"
```

**Correct upgrade flow:**
```sql
-- 1. Cancel old subscription
UPDATE subscriptions 
SET status = 'canceled', canceled_at = NOW()
WHERE account_id = 'acc_123' AND status = 'active';

-- 2. Create new subscription
INSERT INTO subscriptions (account_id, plan_type, status, ...)
VALUES ('acc_123', 'pro', 'active', ...);
```

---

## **📍 NEW SECTION: Phase 3 - Cloudflare Worker Cron Jobs**

**Add this as a new section after Phase 2 in your document:**

---

# **PHASE 3: CLOUDFLARE WORKER CRON JOBS**

## **🕐 Cron Configuration**

### **wrangler.toml Setup**

Add scheduled triggers to your Cloudflare Worker configuration:

```toml
# wrangler.toml
name = "oslira-worker"
main = "src/index.ts"

[triggers]
crons = [
  "0 2 * * *",      # Daily cleanup - 2 AM UTC
  "0 3 1 * *"       # Monthly rollup - 3 AM UTC on 1st of month
]
```

**Testing cron triggers locally:**
```bash
# Trigger daily cleanup
npx wrangler dev --test-scheduled

# In another terminal
curl "http://localhost:8787/__scheduled?cron=0+2+*+*+*"
```

---

## **📅 Daily Cleanup Job**

**Purpose:** 
- Delete AI usage logs older than 90 days
- Hard-delete soft-deleted records older than 30 days
- Prevent database bloat

**Implementation:**

```javascript
/**
 * Daily Cleanup Cron Job
 * Runs at 2 AM UTC daily
 */
export async function dailyCleanup(env, requestId) {
  const supabaseUrl = await getApiKey('SUPABASE_URL', env, env.APP_ENV);
  const serviceRole = await getApiKey('SUPABASE_SERVICE_ROLE', env, env.APP_ENV);
  
  const headers = {
    'apikey': serviceRole,
    'Authorization': `Bearer ${serviceRole}`,
    'Content-Type': 'application/json'
  };
  
  logger('info', 'Starting daily cleanup', { requestId });
  
  try {
    // 1. Delete AI logs older than 90 days
    const aiLogsDate = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000).toISOString();
    const aiLogsRes = await fetch(
      `${supabaseUrl}/rest/v1/ai_usage_logs?created_at=lt.${aiLogsDate}`,
      { method: 'DELETE', headers }
    );
    
    const aiLogsDeleted = aiLogsRes.headers.get('content-range')?.split('/')[1] || '0';
    logger('info', `Deleted ${aiLogsDeleted} old AI logs`, { requestId });
    
    // 2. Hard-delete leads soft-deleted 30+ days ago
    const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString();
    const leadsRes = await fetch(
      `${supabaseUrl}/rest/v1/leads?deleted_at=lt.${thirtyDaysAgo}`,
      { method: 'DELETE', headers }
    );
    
    const leadsDeleted = leadsRes.headers.get('content-range')?.split('/')[1] || '0';
    logger('info', `Hard-deleted ${leadsDeleted} old leads`, { requestId });
    
    // NOTE: Analyses CASCADE delete automatically (no separate query needed)
    
    // 3. Hard-delete business profiles soft-deleted 30+ days ago
    const profilesRes = await fetch(
      `${supabaseUrl}/rest/v1/business_profiles?deleted_at=lt.${thirtyDaysAgo}`,
      { method: 'DELETE', headers }
    );
    
    const profilesDeleted = profilesRes.headers.get('content-range')?.split('/')[1] || '0';
    logger('info', `Hard-deleted ${profilesDeleted} old business profiles`, { requestId });
    
    // 4. Hard-delete accounts soft-deleted 30+ days ago
    const accountsRes = await fetch(
      `${supabaseUrl}/rest/v1/accounts?deleted_at=lt.${thirtyDaysAgo}`,
      { method: 'DELETE', headers }
    );
    
    const accountsDeleted = accountsRes.headers.get('content-range')?.split('/')[1] || '0';
    logger('info', `Hard-deleted ${accountsDeleted} old accounts`, { requestId });
    
    logger('info', 'Daily cleanup completed successfully', {
      requestId,
      aiLogs: aiLogsDeleted,
      leads: leadsDeleted,
      profiles: profilesDeleted,
      accounts: accountsDeleted
    });
    
    return {
      success: true,
      deleted: {
        aiLogs: parseInt(aiLogsDeleted),
        leads: parseInt(leadsDeleted),
        profiles: parseInt(profilesDeleted),
        accounts: parseInt(accountsDeleted)
      }
    };
    
  } catch (error) {
    logger('error', 'Daily cleanup failed', {
      requestId,
      error: error.message,
      stack: error.stack
    });
    throw error;
  }
}
```

---

## **📊 Monthly Rollup Job**

**Purpose:**
- Finalize previous month's usage summaries
- Grant monthly credits to all active subscriptions
- Generate monthly platform metrics

**Implementation:**

```javascript
/**
 * Monthly Rollup Cron Job
 * Runs at 3 AM UTC on the 1st of each month
 */
export async function monthlyRollup(env, requestId) {
  const supabaseUrl = await getApiKey('SUPABASE_URL', env, env.APP_ENV);
  const serviceRole = await getApiKey('SUPABASE_SERVICE_ROLE', env, env.APP_ENV);
  
  const headers = {
    'apikey': serviceRole,
    'Authorization': `Bearer ${serviceRole}`,
    'Content-Type': 'application/json'
  };
  
  logger('info', 'Starting monthly rollup', { requestId });
  
  try {
    // 1. Finalize last month's usage summaries
    const lastMonthEnd = new Date();
    lastMonthEnd.setDate(0); // Last day of previous month
    lastMonthEnd.setHours(23, 59, 59, 999);
    
    await fetch(
      `${supabaseUrl}/rest/v1/account_usage_summary?period_end=eq.${lastMonthEnd.toISOString().split('T')[0]}`,
      {
        method: 'PATCH',
        headers,
        body: JSON.stringify({ is_finalized: true })
      }
    );
    
    logger('info', 'Finalized last month usage summaries', { requestId });
    
    // 2. Get all active subscriptions that need renewal
    const now = new Date();
    const subsRes = await fetch(
      `${supabaseUrl}/rest/v1/rpc/get_renewable_subscriptions`,
      {
        method: 'POST',
        headers,
        body: JSON.stringify({})
      }
    );
    
    if (!subsRes.ok) {
      throw new Error(`Failed to get renewable subscriptions: ${subsRes.status}`);
    }
    
    const subscriptions = await subsRes.json();
    
    logger('info', `Found ${subscriptions.length} subscriptions to renew`, { requestId });
    
    // 3. Grant credits to each active subscription
    let renewedCount = 0;
    
    for (const sub of subscriptions) {
      try {
        // Call deduct_credits with positive amount (grant)
        await fetch(
          `${supabaseUrl}/rest/v1/rpc/deduct_credits`,
          {
            method: 'POST',
            headers,
            body: JSON.stringify({
              p_account_id: sub.account_id,
              p_amount: sub.credits_per_month,  // Positive = grant
              p_transaction_type: 'subscription_renewal',
              p_description: `Monthly credit renewal - ${sub.plan_type} plan`,
              p_metadata: {
                subscription_id: sub.id,
                plan_type: sub.plan_type,
                renewal_date: now.toISOString()
              }
            })
          }
        );
        
        // Update subscription period
        const newPeriodEnd = new Date(sub.current_period_end);
        newPeriodEnd.setMonth(newPeriodEnd.getMonth() + 1);
        
        await fetch(
          `${supabaseUrl}/rest/v1/subscriptions?id=eq.${sub.id}`,
          {
            method: 'PATCH',
            headers,
            body: JSON.stringify({
              current_period_start: sub.current_period_end,
              current_period_end: newPeriodEnd.toISOString()
            })
          }
        );
        
        renewedCount++;
        
      } catch (error) {
        logger('error', 'Failed to renew subscription', {
          requestId,
          subscription_id: sub.id,
          account_id: sub.account_id,
          error: error.message
        });
        // Continue with other subscriptions
      }
    }
    
    logger('info', 'Monthly rollup completed successfully', {
      requestId,
      subscriptions_renewed: renewedCount
    });
    
    return {
      success: true,
      renewed: renewedCount
    };
    
  } catch (error) {
    logger('error', 'Monthly rollup failed', {
      requestId,
      error: error.message,
      stack: error.stack
    });
    throw error;
  }
}
```

---

## **🔧 Helper RPC Function for Monthly Renewal**

**Add this function to Supabase (Phase 1 section):**

```sql
CREATE OR REPLACE FUNCTION get_renewable_subscriptions()
RETURNS TABLE (
  id uuid,
  account_id uuid,
  plan_type text,
  credits_per_month integer,
  current_period_end timestamptz
) AS $$
  SELECT 
    s.id,
    s.account_id,
    s.plan_type,
    p.credits_per_month,
    s.current_period_end
  FROM subscriptions s
  JOIN plans p ON p.id = s.plan_type
  JOIN accounts a ON a.id = s.account_id
  WHERE s.status = 'active'
    AND s.current_period_end < NOW()
    AND a.deleted_at IS NULL
    AND a.is_suspended = false
$$ LANGUAGE sql STABLE;
```

**What it does:** Returns all active subscriptions that need monthly credit renewal

---

## **🎯 Main Worker Export**

**Update your main worker file:**

```javascript
// src/index.ts
import { Hono } from 'hono';
import { dailyCleanup, monthlyRollup } from './cron/scheduled-jobs';

const app = new Hono();

// ... your existing routes ...

export default {
  // HTTP requests
  fetch: app.fetch,
  
  // Scheduled tasks
  async scheduled(event, env, ctx) {
    const requestId = crypto.randomUUID();
    
    try {
      // Daily cleanup (2 AM UTC)
      if (event.cron === '0 2 * * *') {
        await dailyCleanup(env, requestId);
      }
      
      // Monthly rollup (1st of month, 3 AM UTC)
      if (event.cron === '0 3 1 * *') {
        await monthlyRollup(env, requestId);
      }
      
    } catch (error) {
      console.error('Cron job failed:', error);
      // Don't throw - let Cloudflare retry
    }
  }
};
```

---

## **📊 Monitoring Cron Jobs**

### **View Cron Logs (Cloudflare Dashboard)**
1. Go to Workers & Pages → Your Worker
2. Click "Logs" tab
3. Filter by: `cron`

### **Manual Trigger (Testing)**
```bash
# Trigger daily cleanup
curl -X POST https://your-worker.workers.dev/__scheduled \
  -H "Content-Type: application/json" \
  -d '{"cron": "0 2 * * *"}'

# Trigger monthly rollup
curl -X POST https://your-worker.workers.dev/__scheduled \
  -H "Content-Type: application/json" \
  -d '{"cron": "0 3 1 * *"}'
```

### **Alert on Failures (Optional)**
```javascript
// Send alert if cron fails
async function alertAdmin(message, details) {
  // Option A: Log to external service
  await fetch('https://your-monitoring-service.com/alerts', {
    method: 'POST',
    body: JSON.stringify({ message, details })
  });
  
  // Option B: Send email via service
  // Option C: Slack webhook
  // Option D: PagerDuty, etc.
}
```

---

## **⚙️ Cron Job Configuration**

### **Credit Renewal Logic**

**Important:** Credits **accumulate** (they don't reset)

```
User has 15 unused credits
Month renews with 25 credits
New balance: 15 + 25 = 40 credits ✅

This is the standard SaaS behavior (roll-over credits)
```

**Why accumulate?**
- User-friendly (don't lose unused credits)
- Encourages annual subscriptions
- Industry standard (AWS, Stripe, etc. all do this)

**If you want to add a cap later:**
```javascript
// In monthlyRollup function, before granting credits:
const currentBalance = await getCurrentBalance(sub.account_id);
const maxBalance = sub.credits_per_month * 3;  // Cap at 3 months

if (currentBalance >= maxBalance) {
  logger('info', 'Skipping renewal - balance at cap', {
    account_id: sub.account_id,
    current: currentBalance,
    max: maxBalance
  });
  continue;  // Don't grant credits
}
```

---

## **✅ Phase 3 Deployment Checklist**

- [ ] Added `[triggers]` section to `wrangler.toml`
- [ ] Created `dailyCleanup()` function in worker
- [ ] Created `monthlyRollup()` function in worker
- [ ] Added `get_renewable_subscriptions()` RPC to Supabase
- [ ] Updated main worker export with `scheduled` handler
- [ ] Tested cron locally with `wrangler dev --test-scheduled`
- [ ] Deployed worker with `wrangler deploy`
- [ ] Verified cron shows in Cloudflare dashboard (Workers → Triggers)
- [ ] Manually triggered test run
- [ ] Checked logs for successful execution
- [ ] Set up monitoring/alerts for failures

---

**END OF PHASE 3**

---

## **📍 ADD TO LAYER 2: Plans Preseeding**

**Insert this in your "Complete Table Definitions" section after the plans table definition:**

⚠️ MINOR INCOHERENCE FOUND
1. Plans Preseeding Section is Cut Off
In the missing docs artifact, this section is incomplete:
markdown## **📍 ADD TO LAYER 2: Plans Preseeding**

**Insert this in your "Complete Table Definitions" section after the plans table definition:**

###
It just stops. You need the actual INSERT statements for placeholder plans.
Here's what's missing:
sql-- Preseed Plans Data
INSERT INTO plans (id, name, credits_per_month, price_cents, stripe_price_id, features) VALUES
  (
    'free', 
    'Free Plan', 
    25, 
    0, 
    NULL, 
    '{"max_businesses": 1, "max_analyses_per_month": 10}'::jsonb
  ),
  (
    'pro', 
    'Pro Plan', 
    500, 
    9700, 
    'REPLACE_WITH_STRIPE_PRICE_ID', 
    '{"max_businesses": 5, "max_analyses_per_month": 1000}'::jsonb
  ),
  (
    'enterprise', 
    'Enterprise Plan', 
    2000, 
    29700, 
    'REPLACE_WITH_STRIPE_PRICE_ID', 
    '{"max_businesses": null, "max_analyses_per_month": null}'::jsonb
  );
Note: The 'REPLACE_WITH_STRIPE_PRICE_ID' placeholders need to be updated with actual Stripe price IDs after you create products in Stripe dashboard.

2. Missing: Initial Admin Setup
Your doc doesn't explain HOW to create the first admin user.
You need to add this to Phase 1:
markdown## **🔧 Creating First Admin User**

**After deploying the database:**
```sql
-- Find your user ID from Supabase auth dashboard
SELECT id, email FROM auth.users WHERE email = 'your-email@example.com';

-- Grant admin access
UPDATE users 
SET is_admin = true 
WHERE id = 'your-user-uuid-from-above';
```

**Or via Supabase Dashboard:**
1. Go to Authentication → Users
2. Click your user
3. Edit user metadata
4. Add custom claim: `is_admin: true`

3. Missing: Environment Variable Setup
Your doc mentions AWS Secrets but doesn't show Cloudflare Worker env setup.
You need to add this to Phase 0:
markdown## **🔧 Cloudflare Worker Environment Setup**

**Add these to your Cloudflare Worker (via Dashboard or wrangler.toml):**
```toml
# wrangler.toml
[vars]
APP_ENV = "production"  # or "staging"
AWS_REGION = "us-east-1"

# Add these as SECRETS (not vars):
# - AWS_ACCESS_KEY_ID
# - AWS_SECRET_ACCESS_KEY
# - SUPABASE_URL
# - SUPABASE_SERVICE_ROLE

# Run these commands:
npx wrangler secret put AWS_ACCESS_KEY_ID
npx wrangler secret put AWS_SECRET_ACCESS_KEY
npx wrangler secret put SUPABASE_URL
npx wrangler secret put SUPABASE_SERVICE_ROLE
```

**Or via Cloudflare Dashboard:**
1. Workers & Pages → Your Worker → Settings → Variables
2. Add each secret under "Environment Variables" (Encrypted)

4. Missing: Stripe Product Creation Guide
Your doc says "create products in Stripe" but doesn't explain HOW.
You need to add this:
markdown## **💳 Stripe Setup Guide**

### **1. Create Products in Stripe Dashboard**

1. Go to: https://dashboard.stripe.com/products
2. Click "Add product"

**Free Plan:**
- Name: `Free Plan`
- Pricing: One-time payment of $0 (or skip - no Stripe product needed for free)

**Pro Plan:**
- Name: `Pro Plan`
- Pricing: Recurring → Monthly → $97.00
- Copy the Price ID (starts with `price_...`)
- Update your `plans` table: `UPDATE plans SET stripe_price_id = 'price_xxx' WHERE id = 'pro';`

**Enterprise Plan:**
- Name: `Enterprise Plan`
- Pricing: Recurring → Monthly → $297.00
- Copy the Price ID
- Update: `UPDATE plans SET stripe_price_id = 'price_yyy' WHERE id = 'enterprise';`

### **2. Create Webhook Endpoint**

1. Go to: https://dashboard.stripe.com/webhooks
2. Add endpoint: `https://your-worker.workers.dev/webhooks/stripe`
3. Select events:
   - `invoice.paid`
   - `invoice.payment_failed`
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
4. Copy webhook signing secret
5. Add to worker: `npx wrangler secret put STRIPE_WEBHOOK_SECRET`

5. Missing: Deployment Order
Your phases are numbered but don't show WHEN to deploy what.
You need to add this:
markdown## **🚀 DEPLOYMENT ORDER**

### **Step 1: Supabase Setup (30 minutes)**
1. Create new Supabase project
2. Copy Phase 0 + Phase 1 + Phase 2 SQL
3. Paste into Supabase SQL Editor → Execute
4. Verify tables created: `SELECT * FROM users;`
5. Create first admin user (see Phase 1)

### **Step 2: Stripe Setup (15 minutes)**
1. Create products (Free, Pro, Enterprise)
2. Copy price IDs
3. Update plans table with Stripe price IDs
4. Create webhook endpoint
5. Copy webhook secret

### **Step 3: AWS Setup (10 minutes)**
1. Create IAM user with Secrets Manager access
2. Store API keys in AWS Secrets Manager (see deploy-aws-secrets.sh)
3. Copy AWS access keys

### **Step 4: Cloudflare Worker Deploy (20 minutes)**
1. Add environment variables to wrangler.toml
2. Add secrets: `wrangler secret put ...`
3. Deploy: `wrangler deploy`
4. Test signup flow
5. Test cron jobs: `wrangler dev --test-scheduled`

### **Step 5: Verification (15 minutes)**
1. Test signup → Should create account + grant 25 credits
2. Test RLS → Non-admin can't see other accounts
3. Test credit deduction → Balance updates correctly
4. Test webhooks → Stripe events recorded
5. Monitor logs for errors

**Total deployment time: ~90 minutes**

📋 MISSING PIECES SUMMARY
Missing ItemCritical?Where to AddPlans INSERT statements🟡 MediumPhase 0 (preseed section)First admin setup🟡 MediumPhase 1 (after RLS)Cloudflare env vars🔴 CriticalPhase 0 (deployment)Stripe setup guide🔴 CriticalNew sectionDeployment order🔴 CriticalNew section at end

everything is handled via aws not via cloudflare env variables
