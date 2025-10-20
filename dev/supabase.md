# 📘 OSLIRA DATABASE - IMPLEMENTATION GUIDE

**Status:** Production Database Deployed  
**Version:** 2.0  
**Last Updated:** 2025-01-XX

---

## 🎯 OVERVIEW

This guide is for developers building against the Oslira production database. The database is **already deployed** and running. This document explains:

- ✅ What tables exist and their relationships
- ✅ How to query data correctly
- ✅ How to perform common operations
- ✅ What functions to call (don't reinvent the wheel)
- ✅ Security rules (RLS) you must follow
- ✅ Common pitfalls and how to avoid them

**Audience:** Frontend developers, backend developers, new team members

---

# 📊 DATABASE STRUCTURE

## **15 Tables in 3 Layers**

### **LAYER 1: Identity (1 table)**
- `users` - User profiles (extends auth.users)

### **LAYER 2: Billing (10 tables)**
- `accounts` - Billable entities (NOT users)
- `account_members` - User-to-account memberships
- `plans` - Subscription tiers (free/pro/agency/enterprise)
- `subscriptions` - Subscription history (append-only)
- `credit_ledger` - All credit transactions (append-only audit log)
- `credit_balances` - Current credit balance (materialized view)
- `account_usage_summary` - Monthly rollup stats
- `platform_metrics_daily` - Admin analytics
- `stripe_invoices` - Billing history
- `webhook_events` - Stripe idempotency tracking

### **LAYER 3: Business Logic (4 tables)**
- `business_profiles` - Business context for AI analysis
- `leads` - Instagram accounts being analyzed
- `analyses` - AI analysis results
- `ai_usage_logs` - API cost tracking (90-day retention)

---

# 🔑 KEY CONCEPTS

## **1. Accounts Own Everything (Not Users)**

```
❌ WRONG MENTAL MODEL:
User has credits → User analyzes leads

✅ CORRECT MENTAL MODEL:
Account has credits → User belongs to Account → User spends Account's credits
```

**Why this matters:**
- Users can belong to MULTIPLE accounts
- Credits belong to ACCOUNTS
- All queries must filter by `account_id`

**Example:**
```javascript
// ❌ WRONG
const leads = await db.query('SELECT * FROM leads WHERE user_id = ?');

// ✅ CORRECT
const leads = await db.query('SELECT * FROM leads WHERE account_id = ?');
```

---

## **2. Two Credit Tables (Don't Use Wrong One)**

### **credit_ledger (Audit Log)**
- **Purpose:** Financial audit trail
- **Never:** UPDATE or DELETE rows
- **Always:** INSERT only
- **Query When:** Showing transaction history, refunds, debugging
- **Don't Query When:** Checking current balance (slow!)

### **credit_balances (Fast Lookup)**
- **Purpose:** Performance (instant balance check)
- **Never:** INSERT or UPDATE manually
- **Auto-Updated:** By trigger when credit_ledger changes
- **Query When:** Checking if user can afford analysis
- **Don't Query When:** Showing transaction history

**Example:**
```javascript
// ❌ WRONG (slow, complex)
const balance = await db.query(`
  SELECT SUM(amount) FROM credit_ledger WHERE account_id = ?
`);

// ✅ CORRECT (instant)
const balance = await db.query(`
  SELECT current_balance FROM credit_balances WHERE account_id = ?
`);
```

---

## **3. Never Directly INSERT into credit_ledger**

### **Always use the `deduct_credits()` function:**
**⚠️ CRITICAL: Always use Worker with service role key for this operation**

This operation must NEVER be called from the frontend. The `deduct_credits()` function is marked `SECURITY DEFINER` which means it bypasses RLS. Only your Cloudflare Worker with the service role key should call this.
```javascript
// ❌ NEVER DO THIS IN FRONTEND
const { data } = await supabase.rpc('deduct_credits', ...);
// User could manipulate params to deduct from other accounts

// ✅ ONLY IN BACKEND WORKER
const { data } = await supabaseAdmin.rpc('deduct_credits', ...);
// Service role + SECURITY DEFINER = safe
```
```javascript
// ❌ WRONG (race conditions, no overdraft protection)
const balance = await getBalance(accountId);
if (balance >= 2) {
  await db.query(`
    INSERT INTO credit_ledger (account_id, amount, ...) 
    VALUES (?, -2, ...)
  `);
}

// ✅ CORRECT (atomic, safe)
try {
  const txId = await db.rpc('deduct_credits', {
    p_account_id: accountId,
    p_amount: -2,  // Negative = deduction
    p_transaction_type: 'analysis',
    p_description: 'Deep analysis of @nike'
  });
  // Success! Credits deducted
} catch (error) {
  // error.message === "Insufficient credits: 1 available, 2 required"
  return { error: 'Not enough credits' };
}
```

**Why function:**
- Locks row (prevents race conditions)
- Validates balance (no overdraft)
- Updates both tables atomically
- Consistent error messages

---

## **4. Append-Only Subscriptions (Never UPDATE plan_type)**

```javascript
// ❌ WRONG (destroys audit trail)
await db.query(`
  UPDATE subscriptions 
  SET plan_type = 'pro' 
  WHERE account_id = ?
`);

// ✅ CORRECT (preserves history)
// 1. Cancel old subscription
await db.query(`
  UPDATE subscriptions 
  SET status = 'canceled', canceled_at = NOW()
  WHERE account_id = ? AND status = 'active'
`);

// 2. Create new subscription
await db.query(`
  INSERT INTO subscriptions (account_id, plan_type, status, ...)
  VALUES (?, 'pro', 'active', ...)
`);
```

**Why:** Full subscription history preserved (free → pro → enterprise)

---

## **5. Soft Deletes (Always Check deleted_at)**

```javascript
// ❌ WRONG (includes deleted records)
const leads = await db.query('SELECT * FROM leads WHERE account_id = ?');

// ✅ CORRECT (active only)
const leads = await db.query(`
  SELECT * FROM leads 
  WHERE account_id = ? 
    AND deleted_at IS NULL
`);
```

**Tables with soft delete:**
- accounts
- business_profiles
- leads
- analyses

**Recovery window:** 30 days (then hard-deleted by cron)

---

## **6. RLS is Enabled (Use Correct Database Client)**

### **From Frontend (Client SDK):**
```javascript
// Uses user's JWT token
// Can only see data from their accounts
const { data } = await supabase
  .from('leads')
  .select('*')
  .eq('account_id', accountId);
// RLS filters automatically
```

### **From Backend (Service Role):**
```javascript
// Bypasses ALL RLS (admin access)
// Used in Cloudflare Worker
const { data } = await supabaseAdmin
  .from('leads')
  .select('*');
// Sees ALL leads from ALL accounts
```

**Never:** Use service role key in frontend code (security breach)

---

# 🔧 COMMON OPERATIONS

## **Operation: Get User's Accounts**
```javascript
async function getUserAccounts(userId) {
  // Frontend uses anon key - RLS filters automatically
  const { data } = await supabase
    .from('accounts')
    .select(`
      id,
      name,
      slug,
      account_members!inner(role),
      credit_balances(current_balance)
    `)
    .eq('account_members.user_id', userId)
    .is('deleted_at', null)
    .eq('is_suspended', false);
  
  // RLS policy ensures user only sees accounts they're a member of
  return data;
}

// Returns:
// [
//   {
//     id: 'acc_123',
//     name: "Hamza's Account",
//     slug: 'hamza-williams',
//     account_members: [{ role: 'owner' }],
//     credit_balances: [{ current_balance: 48 }]
//   }
// ]

// ❌ User cannot see accounts they don't belong to
// ❌ Anonymous users get empty array (RLS blocks access)
```

---

## **Operation: Check Credit Balance**

```javascript
async function getCredits(accountId) {
  const { data } = await supabase
    .from('credit_balances')
    .select('current_balance')
    .eq('account_id', accountId)
    .single();
  
  return data?.current_balance || 0;
}
```

---

## **Operation: Deduct Credits (Start Analysis)**

```javascript
async function startAnalysis(accountId, leadId, analysisType) {
  const creditsNeeded = analysisType === 'deep' ? 2 : 1;
  
  // 1. Check for duplicate in-progress analysis
  const { data: existing } = await supabase
    .from('analyses')
    .select('id')
    .eq('lead_id', leadId)
    .in('status', ['pending', 'processing'])
    .is('deleted_at', null)
    .maybeSingle();
  
  if (existing) {
    return { error: 'Analysis already in progress' };
  }
  
  // 2. Deduct credits (atomic, safe)
  try {
    const { data: txId, error } = await supabaseAdmin.rpc('deduct_credits', {
      p_account_id: accountId,
      p_amount: -creditsNeeded,
      p_transaction_type: 'analysis',
      p_description: `${analysisType} analysis of lead ${leadId}`
    });
    
    if (error) throw error;
    
  } catch (error) {
    return { error: 'Insufficient credits' };
  }
  
  // 3. Create analysis record
  const { data: analysis } = await supabaseAdmin
    .from('analyses')
    .insert({
      lead_id: leadId,
      account_id: accountId,
      analysis_type: analysisType,
      status: 'pending',
      credits_charged: creditsNeeded
    })
    .select()
    .single();
  
  // 4. Queue for background processing
  await queueAnalysisJob(analysis.id);
  
  return { success: true, analysisId: analysis.id };
}
```

---

## **Operation: Get Leads with Latest Analysis**

```javascript
async function getLeadsWithScores(accountId, businessProfileId) {
  const { data } = await supabase
    .from('leads')
    .select(`
      *,
      latest_analysis:analyses!lead_id(
        id,
        overall_score,
        niche_fit_score,
        completed_at
      )
    `)
    .eq('account_id', accountId)
    .eq('business_profile_id', businessProfileId)
    .is('deleted_at', null)
    .order('created_at', { 
      foreignTable: 'analyses', 
      ascending: false 
    })
    .limit(1, { foreignTable: 'analyses' });
  
  return data;
}

// Returns:
// [
//   {
//     id: 'lead_123',
//     instagram_username: 'nike',
//     follower_count: 250000000,
//     latest_analysis: {
//       id: 'analysis_456',
//       overall_score: 85,
//       niche_fit_score: 90,
//       completed_at: '2025-01-15T10:30:00Z'
//     }
//   }
// ]
```

---

## **Operation: Create New Lead (or Update Existing)**

```javascript
async function upsertLead(accountId, businessProfileId, instagramData) {
  // Check if lead exists
  const { data: existing } = await supabase
    .from('leads')
    .select('id')
    .eq('account_id', accountId)
    .eq('business_profile_id', businessProfileId)
    .eq('instagram_username', instagramData.username)
    .is('deleted_at', null)
    .maybeSingle();
  
  if (existing) {
    // Update existing lead (profile may have changed)
    const { data } = await supabase
      .from('leads')
      .update({
        display_name: instagramData.displayName,
        follower_count: instagramData.followers,
        bio: instagramData.bio,
        profile_pic_url: instagramData.profilePicUrl,
        last_analyzed_at: new Date().toISOString()
      })
      .eq('id', existing.id)
      .select()
      .single();
    
    return { leadId: data.id, isNew: false };
  }
  
  // Create new lead
  const { data } = await supabase
    .from('leads')
    .insert({
      account_id: accountId,
      business_profile_id: businessProfileId,
      instagram_username: instagramData.username,
      display_name: instagramData.displayName,
      follower_count: instagramData.followers,
      bio: instagramData.bio,
      profile_pic_url: instagramData.profilePicUrl,
      first_analyzed_at: new Date().toISOString(),
      last_analyzed_at: new Date().toISOString()
    })
    .select()
    .single();
  
  return { leadId: data.id, isNew: true };
}
```

---

## **Operation: Get Transaction History**

```javascript
async function getCreditHistory(accountId, limit = 50) {
  const { data } = await supabase
    .from('credit_ledger')
    .select(`
      *,
      created_by_user:users(full_name)
    `)
    .eq('account_id', accountId)
    .order('created_at', { ascending: false })
    .limit(limit);
  
  return data;
}

// Returns:
// [
//   {
//     id: 'tx_123',
//     amount: -2,
//     balance_after: 48,
//     transaction_type: 'analysis',
//     description: 'Deep analysis of @nike',
//     created_at: '2025-01-15T10:30:00Z',
//     created_by_user: { full_name: 'Hamza Williams' }
//   },
//   {
//     id: 'tx_124',
//     amount: 100,
//     balance_after: 50,
//     transaction_type: 'subscription_renewal',
//     description: 'Monthly Pro renewal',
//     created_at: '2025-01-01T03:00:00Z',
//     created_by_user: null  // System transaction
//   }
// ]
```

---

## **Operation: Soft Delete Lead**

```javascript
async function deleteLead(leadId) {
  // 1. Soft delete lead
  await supabase
    .from('leads')
    .update({ deleted_at: new Date().toISOString() })
    .eq('id', leadId);
  
  // 2. Soft delete related analyses
  await supabase
    .from('analyses')
    .update({ deleted_at: new Date().toISOString() })
    .eq('lead_id', leadId);
  
  // Note: User keeps their credits
  // credit_ledger preserves audit (lead_id → NULL on hard delete)
}
```

---

## **Operation: Restore Deleted Lead**

```javascript
async function restoreLead(leadId) {
  // 1. Check if within recovery window (30 days)
  const { data: lead } = await supabase
    .from('leads')
    .select('deleted_at')
    .eq('id', leadId)
    .single();
  
  const deletedDate = new Date(lead.deleted_at);
  const daysSinceDelete = (Date.now() - deletedDate) / (1000 * 60 * 60 * 24);
  
  if (daysSinceDelete > 30) {
    return { error: 'Lead permanently deleted (>30 days)' };
  }
  
  // 2. Restore lead
  await supabase
    .from('leads')
    .update({ deleted_at: null })
    .eq('id', leadId);
  
  // 3. Restore analyses
  await supabase
    .from('analyses')
    .update({ deleted_at: null })
    .eq('lead_id', leadId);
  
  return { success: true };
}
```

---

# 🎣 WEBHOOK HANDLERS

## **Webhook: invoice.paid**

```javascript
export async function handleInvoicePaid(invoice) {
  // 1. Find account
  const { data: sub } = await supabaseAdmin
    .from('subscriptions')
    .select('account_id')
    .eq('stripe_customer_id', invoice.customer)
    .single();
  
  if (!sub) {
    console.error('No account found for customer:', invoice.customer);
    return;
  }
  
  // 2. Store invoice record (for support team)
  await supabaseAdmin
    .from('stripe_invoices')
    .insert({
      account_id: sub.account_id,
      stripe_invoice_id: invoice.id,
      stripe_subscription_id: invoice.subscription,
      amount_cents: invoice.amount_paid,
      status: 'paid',
      paid_at: new Date(invoice.status_transitions.paid_at * 1000)
    });
  
  console.log('Invoice recorded:', invoice.id);
}
```

---

## **Webhook: invoice.payment_failed**

```javascript
export async function handleInvoiceFailed(invoice) {
  // 1. Find account
  const { data: sub } = await supabaseAdmin
    .from('subscriptions')
    .select('account_id')
    .eq('stripe_customer_id', invoice.customer)
    .single();
  
  if (!sub) return;
  
  // 2. Store failed invoice
  await supabaseAdmin
    .from('stripe_invoices')
    .insert({
      account_id: sub.account_id,
      stripe_invoice_id: invoice.id,
      amount_cents: invoice.amount_due,
      status: 'uncollectible',
      failed_at: new Date(),
      failure_reason: invoice.last_finalization_error?.message
    });
  
  // 3. Update subscription status
  await supabaseAdmin
    .from('subscriptions')
    .update({ status: 'past_due' })
    .eq('stripe_customer_id', invoice.customer)
    .eq('status', 'active');
  
  // 4. Send notification email (optional)
  // await sendPaymentFailedEmail(sub.account_id);
}
```

---

## **Webhook: customer.subscription.created**

```javascript
export async function handleSubscriptionCreated(subscription) {
  // User upgraded from free to paid
  
  // 1. Update existing subscription record
  await supabaseAdmin
    .from('subscriptions')
    .update({
      stripe_subscription_id: subscription.id,
      stripe_price_id: subscription.items.data[0].price.id,
      status: 'active'
    })
    .eq('stripe_customer_id', subscription.customer)
    .eq('status', 'pending_stripe_confirmation');
  
  console.log('Subscription confirmed:', subscription.id);
}
```

---

# 🕐 CRON JOBS

## **Daily Cleanup (2 AM UTC)**

```javascript
// Cloudflare Worker scheduled trigger
export async function dailyCleanup() {
  // 1. Delete AI logs older than 90 days
  await supabaseAdmin
    .from('ai_usage_logs')
    .delete()
    .lt('created_at', new Date(Date.now() - 90 * 24 * 60 * 60 * 1000));
  
  // 2. Hard-delete soft-deleted records (30+ days old)
  const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
  
  await supabaseAdmin
    .from('leads')
    .delete()
    .lt('deleted_at', thirtyDaysAgo);
  
  await supabaseAdmin
    .from('business_profiles')
    .delete()
    .lt('deleted_at', thirtyDaysAgo);
  
  await supabaseAdmin
    .from('accounts')
    .delete()
    .lt('deleted_at', thirtyDaysAgo);
}
```

---

## **Monthly Rollup (1st of month, 3 AM UTC)**

```javascript
export async function monthlyRollup() {
  // 1. Finalize last month's usage summaries
  const lastMonth = new Date();
  lastMonth.setMonth(lastMonth.getMonth() - 1);
  const lastMonthEnd = new Date(lastMonth.getFullYear(), lastMonth.getMonth() + 1, 0);
  
  await supabaseAdmin
    .from('account_usage_summary')
    .update({ is_finalized: true })
    .eq('period_end', lastMonthEnd.toISOString().split('T')[0])
    .eq('is_finalized', false);
  
  // 2. Get all renewable subscriptions
  const { data: subscriptions } = await supabaseAdmin
    .rpc('get_renewable_subscriptions');
  
  // 3. Grant credits to each
  for (const sub of subscriptions) {
    try {
      // Grant credits
      await supabaseAdmin.rpc('deduct_credits', {
        p_account_id: sub.account_id,
        p_amount: sub.credits_per_month,
        p_transaction_type: 'subscription_renewal',
        p_description: `Monthly ${sub.plan_type} renewal`
      });
      
      // Update subscription period
      const newPeriodEnd = new Date(sub.current_period_end);
      newPeriodEnd.setMonth(newPeriodEnd.getMonth() + 1);
      
      await supabaseAdmin
        .from('subscriptions')
        .update({
          current_period_start: sub.current_period_end,
          current_period_end: newPeriodEnd.toISOString()
        })
        .eq('id', sub.id);
      
    } catch (error) {
      console.error(`Failed to renew ${sub.id}:`, error);
    }
  }
}
```

---

# 🚨 COMMON PITFALLS

## **Pitfall #1: Querying Wrong Credit Table**

```javascript
// ❌ SLOW (scans entire ledger)
const balance = await db.query(`
  SELECT SUM(amount) FROM credit_ledger WHERE account_id = ?
`);

// ✅ FAST (single row lookup)
const balance = await db.query(`
  SELECT current_balance FROM credit_balances WHERE account_id = ?
`);
```

---

## **Pitfall #2: Race Condition on Credit Deduction**

```javascript
// ❌ WRONG (two analyses can start simultaneously)
const balance = await getBalance(accountId);
if (balance >= 2) {
  await insertCreditLedger(accountId, -2);
  await createAnalysis();
}

// ✅ CORRECT (atomic, locked)
await db.rpc('deduct_credits', { p_account_id, p_amount: -2, ... });
```

---

## **Pitfall #3: Forgetting deleted_at Filter**

```javascript
// ❌ WRONG (shows deleted leads)
SELECT * FROM leads WHERE account_id = ?

// ✅ CORRECT
SELECT * FROM leads WHERE account_id = ? AND deleted_at IS NULL
```

---

## **Pitfall #4: Using anon Key for Admin Operations**

```javascript
// ❌ WRONG (RLS blocks this)
const supabase = createClient(SUPABASE_URL, ANON_KEY);
await supabase.from('webhook_events').insert(...);
// ERROR: RLS policy violation

// ✅ CORRECT (bypass RLS)
const supabaseAdmin = createClient(SUPABASE_URL, SERVICE_ROLE_KEY);
await supabaseAdmin.from('webhook_events').insert(...);
```

---

## **Pitfall #5: Not Checking for In-Progress Analysis**

```javascript
// ❌ WRONG (user charged twice if they click fast)
await deductCredits(accountId, -2);
await createAnalysis(leadId);

// ✅ CORRECT (check first)
const existing = await db.query(`
  SELECT id FROM analyses 
  WHERE lead_id = ? AND status IN ('pending', 'processing')
`);
if (existing.length > 0) {
  return { error: 'Analysis already in progress' };
}
```
---

## **Pitfall #6: Using Service Role from Frontend**
```javascript
// ❌ CRITICAL SECURITY HOLE (exposes service key)
const supabase = createClient(SUPABASE_URL, SERVICE_ROLE_KEY);
// If this code is in frontend, ANYONE can steal the key from browser

// ✅ CORRECT - Two separate clients
// Frontend (React/Vue/etc):
const supabase = createClient(SUPABASE_URL, ANON_KEY);
// User can only access their data via RLS

// Backend Worker (Cloudflare):
const supabaseAdmin = createClient(SUPABASE_URL, SERVICE_ROLE_KEY);
// Admin access, bypasses RLS
```

**Why this matters:**
- Service role key in frontend = complete database access for attackers
- Attackers can read ALL accounts' data
- Attackers can modify financial records
- This is the #1 most dangerous Supabase mistake

**Rule:** Service role key = Backend/Worker ONLY. Frontend = Anon key ONLY.
---



# 🔐 SECURITY RULES

## **RLS is Enabled - What This Means:**

### **Frontend Queries (Authenticated Users Only):**
- ✅ Can see: Their own account's data only
- ✅ Can modify: Their own account's data only (WITH CHECK enforced)
- ❌ Cannot see: Other accounts' data
- ❌ Cannot see: Deleted records
- ❌ Cannot see: Suspended accounts
- ❌ Cannot access: If not logged in (anon users blocked)

### **Backend Queries (Service Role via Worker):**
- ✅ Can see: Everything (all accounts)
- ✅ Can modify: Everything (bypasses RLS)
- ✅ Can see: Deleted records
- ✅ Can see: Suspended accounts
- ⚠️ Must validate permissions in Worker code
- ⚠️ Never expose service role key to frontend

---

## **Tables with RLS Enabled (Authenticated Users Only):**

**Full Access (SELECT/INSERT/UPDATE/DELETE):**
- accounts (filtered by user's account membership)
- account_members (filtered by user's account membership)
- leads (filtered by user's account membership)
- analyses (filtered by user's account membership)
- business_profiles (filtered by user's account membership)

**Read-Only Access (SELECT only):**
- credit_ledger (user can view their transactions, Worker writes via RPC)
- credit_balances (user can view balance, Worker writes via trigger)
- account_usage_summary (user can view stats, Worker calculates)
- subscriptions (user can view history, Stripe webhooks write)
- stripe_invoices (user can view invoices, Stripe webhooks write)
- plans (all authenticated users can read, Worker writes)

**No User Access (Service Role Only via Worker):**
- webhook_events (system logging)
- ai_usage_logs (cost tracking)
- platform_metrics_daily (admin analytics)

---

## **Security Features Enforced:**

✅ **Multi-tenant isolation:** Users cannot access other accounts' data  
✅ **WITH CHECK clauses:** Users cannot insert/update data for other accounts  
✅ **No anonymous access:** Must be logged in to access any data  
✅ **SECURITY DEFINER functions:** Credit operations bypass RLS safely  
✅ **Soft delete filtering:** Deleted records hidden from queries  
✅ **Performance optimized:** auth.uid() wrapped in subqueries (100x faster)

---

# 🛡️ SECURITY AUDIT CHECKLIST

Run these queries periodically to verify security:

## **1. Verify All Functions Have SECURITY DEFINER**
```sql
SELECT 
  proname,
  CASE WHEN prosecdef THEN '✅ SECURE' ELSE '❌ FIX' END
FROM pg_proc
WHERE proname IN ('deduct_credits', 'update_credit_balance', 'get_renewable_subscriptions')
  AND pronamespace = 'public'::regnamespace;
-- All should show ✅ SECURE
```

## **2. Verify All Policies Use Authenticated Role**
```sql
SELECT tablename, policyname, roles
FROM pg_policies
WHERE schemaname = 'public'
  AND 'public' = ANY(roles) OR 'anon' = ANY(roles);
-- Should return 0 rows (except plans table which is public read)
```

## **3. Verify INSERT/UPDATE Policies Have WITH CHECK**
```sql
SELECT tablename, policyname
FROM pg_policies
WHERE schemaname = 'public'
  AND cmd IN ('INSERT', 'UPDATE', 'ALL')
  AND with_check IS NULL
  AND tablename IN (
    'accounts', 'account_members', 'business_profiles', 
    'leads', 'analyses'
  );
-- Should return 0 rows
```

## **4. Check for Views Without RLS**
```sql
SELECT viewname
FROM pg_views 
WHERE schemaname = 'public';
-- Verify each view has security_invoker = on
```

## **5. Verify No Exposed HTTP Extension**
```sql
SELECT routine_name, grantee
FROM information_schema.routine_privileges
WHERE routine_schema = 'extensions' 
  AND routine_name LIKE 'http%'
  AND grantee IN ('public', 'anon', 'authenticated');
-- Should return 0 rows
```

**Run this audit monthly or after any schema changes.**

---

# 📋 QUICK REFERENCE

## **Database Functions (Use These, Don't Reimplement):**

| Function | Purpose | Usage |
|----------|---------|-------|
| `deduct_credits()` | Add/remove credits atomically | `db.rpc('deduct_credits', {p_account_id, p_amount, ...})` |
| `generate_slug()` | Create unique account slug | `db.rpc('generate_slug', {input_text: 'Hamza Williams'})` |
| `get_renewable_subscriptions()` | Find subs needing renewal | Called by monthly cron |

---

## **Pricing:**

| Plan | Credits/Month | Price | Max Businesses |
|------|---------------|-------|----------------|
| Free | 25 | $0 | 1 |
| Pro | 100 | $30 | 3 |
| Agency | 300 | $80 | 10 |
| Enterprise | 1500 | $300 | Unlimited |

**Credit Costs:**
- Light analysis: 1 credit
- Deep analysis: 2 credits

**Credit Rollover:** Yes (accumulate, no expiration)

---

## **Important Constraints:**

- ✅ One active subscription per account
- ✅ Same Instagram username allowed for different businesses
- ✅ Multiple analyses allowed per lead (historical tracking)
- ✅ Duplicate webhook events blocked (idempotency)
- ✅ Credit balance cannot go negative (enforced by function)

---

## **Data Retention:**

| Data | Retention | Reason |
|------|-----------|--------|
| credit_ledger | Forever | Audit/legal |
| subscriptions | Forever | Audit/legal |
| ai_usage_logs | 90 days | Performance metrics |
| Soft deletes | 30 days | Recovery window |

---

# 🆘 TROUBLESHOOTING

## **"Insufficient credits" Error**

```javascript
// Check actual balance
const { data } = await supabase
  .from('credit_balances')
  .select('current_balance')
  .eq('account_id', accountId)
  .single();

console.log('Balance:', data.current_balance);

// Check recent transactions
const { data: history } = await supabase
  .from('credit_ledger')
  .select('*')
  .eq('account_id', accountId)
  .order('created_at', { ascending: false })
  .limit(10);

console.log('Recent transactions:', history);
```

---

## **"Analysis stuck in 'processing'"**

```sql
-- Find stuck analyses (manual cleanup)
SELECT * FROM analyses 
WHERE status = 'processing' 
  AND started_at < NOW() - INTERVAL '10 minutes';

-- Mark as failed and refund credits
UPDATE analyses SET status = 'failed', completed_at = NOW() WHERE id = ?;

-- Call deduct_credits() with positive amount to refund
```

---

## **"User can't see their data"**

```javascript
// Check if user is member of account
const { data } = await supabase
  .from('account_members')
  .select('account_id, role')
  .eq('user_id', userId);

console.log('User accounts:', data);

// Check if account is suspended/deleted
const { data: accounts } = await supabase
  .from('accounts')
  .select('id, is_suspended, deleted_at')
  .in('id', data.map(m => m.account_id));

console.log('Account statuses:', accounts);
```

---

**END OF IMPLEMENTATION GUIDE**

For SQL schema details, see original master plan documents.

cloudflare handles cron jobs
