# OSLIRA SUPABASE - PRODUCTION REFERENCE

**Version:** 2.1 | **Updated:** 2025-02-11 | **Status:** Production

---

## 🏗️ ARCHITECTURE

**16 Tables, 3 Layers:**
- **Identity (1):** users
- **Billing (10):** accounts, account_members, subscriptions, plans, credit_ledger, credit_balances, account_usage_summary, stripe_invoices, webhook_events, platform_metrics_daily
- **Business (5):** business_profiles, leads, analyses, ai_usage_logs, refresh_tokens

**Golden Rule:** Accounts own everything. Users belong to accounts. Credits belong to accounts. Always filter by `account_id`.

---

## 📊 TABLE REFERENCE

### **users** (11 columns, NO RLS)
- `id` (uuid, PK) → Matches auth.users
- `email` (text, UNIQUE, NOT NULL)
- `full_name`, `avatar_url` (text)
- `is_admin`, `is_suspended` (bool, default false)
- `last_seen_at`, `suspended_at` (timestamptz)

**Note:** Onboarding fields moved to business_profiles

---

### **accounts** (12 columns, RLS enabled)
- `id` (uuid, PK), `owner_id` (uuid, FK → users, NOT NULL)
- `name` (text, NOT NULL), `slug` (text, UNIQUE)
- `stripe_customer_id` (text, UNIQUE, format: cus_%)
- `is_suspended` (bool, default false)
- `deleted_at` (timestamptz) - Soft delete

**Key Indexes:**
- `idx_accounts_active` (WHERE deleted_at IS NULL AND is_suspended = false)
- `idx_accounts_stripe_customer_id`

---

### **account_members** (7 columns, RLS enabled)
- `account_id`, `user_id` (uuid, FK, NOT NULL)
- `role` (text, default 'member') - owner/admin/member/viewer
- `invited_by` (uuid, FK → users)
- UNIQUE(account_id, user_id)

**Critical:** This table defines access. ALL queries MUST filter by account_id via `user_account_ids()`.

---

### **business_profiles** (17 columns, RLS enabled)
- `id` (uuid, PK), `account_id` (uuid, FK → accounts, NOT NULL)
- `business_one_liner` (text) - Generated 1-line pitch
- `ideal_customer_profile` (jsonb) - ICP data
- `business_context` (jsonb) - Full context for AI
- `business_summary_generated` (text) - 4-sentence AI summary
- `signature_name` (text) - First name for outreach
- `full_name` (text) - User's full name
- `onboarding_completed` (bool, default false)
- `onboarding_completed_at` (timestamptz)
- `deleted_at` (timestamptz)

**Key Change:** Identity fields (signature_name, full_name, onboarding) now live HERE, not in users table.

---

### **leads** (20 columns, RLS enabled)
- `id` (uuid, PK), `account_id`, `business_profile_id` (uuid, FK, NOT NULL)
- `instagram_username` (text, NOT NULL)
- `display_name`, `bio`, `profile_pic_url`, `profile_url` (text)
- `follower_count`, `following_count`, `post_count` (int)
- `is_verified`, `is_private`, `is_business_account` (bool)
- `platform` (text, default 'instagram')
- `first_analyzed_at`, `last_analyzed_at` (timestamptz)
- `deleted_at` (timestamptz)

**UNIQUE:** (account_id, business_profile_id, instagram_username) - Same IG user allowed across different businesses

---

### **analyses** (21 columns, RLS enabled)
- `id` (uuid, PK), `lead_id`, `account_id`, `business_profile_id` (uuid, FK, NOT NULL)
- `analysis_type` (text, NOT NULL) - light/deep
- `status` (text, default 'pending') - pending/processing/completed/failed
- `ai_response` (jsonb) - Full AI output
- `overall_score`, `niche_fit_score`, `engagement_score` (int, 0-100)
- `confidence_level` (numeric)
- `credits_charged`, `total_cost_cents` (int)
- `model_used`, `analysis_version` (text)
- `processing_duration_ms` (int)
- `started_at`, `completed_at`, `deleted_at` (timestamptz)

**Key Index:** `idx_analyses_in_progress` (WHERE status IN ('pending', 'processing') AND deleted_at IS NULL)

---

### **credit_ledger** (11 columns, RLS SELECT only)
- `id` (uuid, PK), `account_id` (uuid, FK, NOT NULL)
- `amount` (int, NOT NULL) - Negative = debit, positive = credit
- `balance_after` (int, NOT NULL, >= 0)
- `transaction_type` (text, NOT NULL) - subscription_renewal/analysis/refund/admin_grant/chargeback/signup_bonus
- `reference_type`, `reference_id` (text, uuid)
- `description`, `metadata` (text, jsonb)
- `created_by` (uuid, FK → users)

**APPEND-ONLY:** Never UPDATE or DELETE. Use `deduct_credits()` function to write.

---

### **credit_balances** (5 columns, RLS SELECT only)
- `account_id` (uuid, PK, FK → accounts)
- `current_balance` (int, NOT NULL, default 0, >= 0)
- `last_transaction_at` (timestamptz)

**Auto-updated by trigger:** `credit_ledger_update_balance` fires on INSERT to credit_ledger.

---

### **subscriptions** (13 columns, RLS SELECT only)
- `id` (uuid, PK), `account_id` (uuid, FK → accounts, NOT NULL)
- `plan_type` (text, NOT NULL, FK → plans.id)
- `price_cents` (int, NOT NULL, >= 0)
- `stripe_customer_id`, `stripe_subscription_id`, `stripe_price_id` (text)
- `status` (text, default 'active') - active/canceled/past_due/paused/pending_stripe_confirmation
- `current_period_start`, `current_period_end` (timestamptz, NOT NULL)
- `canceled_at` (timestamptz)

**UNIQUE:** One active subscription per account (idx_one_active_subscription)

---

### **plans** (8 columns, RLS SELECT only)
- `id` (text, PK) - 'free', 'pro', 'agency', 'enterprise'
- `name` (text, NOT NULL)
- `credits_per_month` (int, NOT NULL, >= 0)
- `price_cents` (int, NOT NULL, >= 0)
- `stripe_price_id` (text)
- `features` (jsonb)
- `is_active` (bool, default true)

**Current Plans:**
- free: 25 credits/mo, $0
- pro: 100 credits/mo, $3000 ($30.00)
- agency: 300 credits/mo, $8000 ($80.00)
- enterprise: 1500 credits/mo, $30000 ($300.00)

---

### **refresh_tokens** (8 columns, RLS enabled)
- `id` (uuid, PK), `token` (text, UNIQUE, NOT NULL)
- `user_id`, `account_id` (uuid, FK, NOT NULL)
- `expires_at` (timestamptz, NOT NULL) - 7 days from creation
- `revoked_at` (timestamptz) - NULL if active
- `replaced_by_token` (text) - For rotation audit

**Constraints:**
- `token_not_expired` - expires_at > created_at
- `revoked_has_timestamp` - Consistency check

---

## 🔑 KEY CONCEPTS

### 1. Accounts Own Everything
```sql
-- ❌ WRONG
SELECT * FROM leads WHERE user_id = ?

-- ✅ CORRECT
SELECT * FROM leads 
WHERE account_id IN (SELECT user_account_ids()) 
  AND deleted_at IS NULL
```

### 2. Two Credit Tables
```sql
-- ❌ SLOW (scans entire ledger)
SELECT SUM(amount) FROM credit_ledger WHERE account_id = ?

-- ✅ FAST (single row)
SELECT current_balance FROM credit_balances WHERE account_id = ?
```

### 3. Always Use deduct_credits()
```javascript
// ❌ NEVER (race conditions, no validation)
INSERT INTO credit_ledger ...

// ✅ ALWAYS (atomic, safe, enforces balance >= 0)
const { data: txId, error } = await supabaseAdmin.rpc('deduct_credits', {
  p_account_id: accountId,
  p_amount: -2, // Negative = deduct
  p_transaction_type: 'analysis',
  p_description: 'Deep analysis of @nike',
  p_metadata: { lead_id: 'uuid' }
});
```

**⚠️ CRITICAL:** Only call from Worker with service role key. Function is SECURITY DEFINER.

### 4. Soft Deletes
```sql
-- ❌ WRONG (includes deleted)
SELECT * FROM leads WHERE account_id = ?

-- ✅ CORRECT
SELECT * FROM leads WHERE account_id = ? AND deleted_at IS NULL
```

**Tables with soft delete:** accounts, business_profiles, leads, analyses (30-day recovery window)

### 5. Append-Only Subscriptions
```javascript
// ❌ NEVER UPDATE plan_type
UPDATE subscriptions SET plan_type = 'pro' ...

// ✅ CORRECT (preserves history)
// 1. Cancel old
UPDATE subscriptions 
SET status = 'canceled', canceled_at = NOW()
WHERE account_id = ? AND status = 'active';

// 2. Create new
INSERT INTO subscriptions (account_id, plan_type, ...) ...
```

---

## 🔒 RLS & SECURITY

### Frontend (Anon Key)
- ✅ See only own account's data (via `user_account_ids()`)
- ❌ Cannot see other accounts
- ❌ Cannot see deleted records (policies enforce deleted_at IS NULL)
- ❌ Blocked if not authenticated

### Backend (Service Role Key)
- ✅ Bypass all RLS (see everything)
- ✅ See deleted records
- ⚠️ Must validate permissions in Worker code
- ⚠️ NEVER expose to frontend (critical security hole)

### Helper Functions
```sql
-- Used in all RLS policies
user_account_ids() → SETOF uuid  -- Returns user's account IDs
is_admin() → boolean              -- Returns true if user is admin
```

### Example Policy
```sql
CREATE POLICY "leads_select" ON leads FOR SELECT
USING (
  is_admin() OR 
  (account_id IN (SELECT user_account_ids()) AND deleted_at IS NULL)
);
```

**Tables WITHOUT RLS:** users (managed by auth.users policies)

---

## 🔧 CRITICAL FUNCTIONS

### deduct_credits()
```sql
deduct_credits(
  p_account_id uuid,
  p_amount integer,           -- Negative = deduct, positive = add
  p_transaction_type text,    -- 'analysis', 'subscription_renewal', etc.
  p_description text,
  p_metadata jsonb DEFAULT '{}'
) RETURNS uuid                -- Returns transaction_id
```
- Atomic (locks credit_balances row)
- Validates balance >= 0
- Updates both credit_ledger and credit_balances
- Throws error if insufficient credits
- SECURITY DEFINER (bypasses RLS)

### create_account_atomic()
```sql
create_account_atomic(
  p_user_id uuid,
  p_email text,
  p_full_name text,
  p_avatar_url text DEFAULT NULL
) RETURNS jsonb
```
- Upserts user
- Creates account + membership (role: 'owner')
- Grants 25 signup credits
- Returns: {user_id, account_id, email, credit_balance, onboarding_completed, is_new_user}
- Idempotent (safe to call multiple times)
- SECURITY DEFINER

---

## 📋 COMMON OPERATIONS

### Get User's Accounts
```javascript
const { data } = await supabase
  .from('accounts')
  .select(`
    *,
    account_members!inner(role),
    credit_balances(current_balance)
  `)
  .eq('account_members.user_id', userId)
  .is('deleted_at', null);
```

### Check Balance
```javascript
const { data } = await supabase
  .from('credit_balances')
  .select('current_balance')
  .eq('account_id', accountId)
  .single();
```

### Start Analysis (Worker Only)
```javascript
// 1. Check duplicate
const { data: existing } = await supabaseAdmin
  .from('analyses')
  .select('id')
  .eq('lead_id', leadId)
  .in('status', ['pending', 'processing'])
  .is('deleted_at', null)
  .maybeSingle();

if (existing) return { error: 'Analysis in progress' };

// 2. Deduct credits (atomic)
const { data: txId, error } = await supabaseAdmin.rpc('deduct_credits', {
  p_account_id: accountId,
  p_amount: -2,
  p_transaction_type: 'analysis',
  p_description: `Deep analysis of ${username}`
});

if (error) return { error: 'Insufficient credits' };

// 3. Create analysis
const { data: analysis } = await supabaseAdmin
  .from('analyses')
  .insert({
    lead_id: leadId,
    account_id: accountId,
    business_profile_id: businessProfileId,
    analysis_type: 'deep',
    status: 'pending',
    credits_charged: 2
  })
  .select()
  .single();
```

---

## 🚨 COMMON PITFALLS

1. **Wrong credit table** → Use credit_balances for checks, not credit_ledger
2. **Race conditions** → Always use deduct_credits() function, never INSERT directly
3. **Forgetting deleted_at** → Always filter deleted_at IS NULL
4. **Using anon key for admin ops** → Webhooks/crons need service role key
5. **Not checking duplicates** → Check for in-progress analysis before charging
6. **Service role in frontend** → NEVER expose service key to browser (critical vulnerability)
7. **Missing account_id filter** → Always filter by account_id from user_account_ids()

---

## 📐 DATA RETENTION

| Data | Retention | Reason |
|------|-----------|--------|
| credit_ledger | Forever | Legal/audit |
| subscriptions | Forever | Legal/audit |
| ai_usage_logs | 90 days | Cost metrics (cron cleanup) |
| Soft deletes | 30 days | Recovery window (cron cleanup) |

---

## 🎯 QUICK REFERENCE

**Plans & Pricing:**
| Plan | Credits/mo | Price | Max Businesses |
|------|------------|-------|----------------|
| Free | 25 | $0 | 1 |
| Pro | 100 | $30 | 3 |
| Agency | 300 | $80 | 10 |
| Enterprise | 1500 | $300 | Unlimited |

**Credit Costs:**
- Light analysis: 1 credit
- Deep analysis: 2 credits

**Credit Rollover:** Yes (accumulate, no expiration)

**Critical Constraints:**
- ✅ One active subscription per account
- ✅ Same Instagram username allowed across different businesses
- ✅ Multiple analyses per lead (historical tracking)
- ✅ Duplicate webhook events blocked (idempotency via stripe_event_id)
- ✅ Credit balance cannot go negative (enforced by function)

---

## 🔍 IMPORTANT CHANGES FROM V1

1. **Onboarding moved:** `onboarding_completed`, `onboarding_completed_at`, `signature_name`, `full_name` now in business_profiles (not users)
2. **Stripe customer ID:** Now on accounts table (not users)
3. **RLS helper functions:** Policies now use `user_account_ids()` and `is_admin()` for better performance
4. **Refresh tokens:** New table for JWT rotation with 7-day expiry
5. **Unique constraints:** Enhanced for data integrity (e.g., one active subscription per account)

---

**END OF REFERENCE** | For schema extraction: see extract_schema.sql
