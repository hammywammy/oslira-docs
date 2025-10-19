# 📘 OSLIRA DATABASE V2 - MASTER PLAN

**Status:** Production-Ready Blueprint  
**Purpose:** Complete database architecture specification (plan only, not implementation)  
**Last Updated:** 2025-01-XX

---

## 🎯 EXECUTIVE SUMMARY

This document defines the complete PostgreSQL database schema for Oslira, a B2B SaaS platform for Instagram lead analysis.

### **What This Document Contains:**
- ✅ **15 tables** with complete column specifications
- ✅ **All constraints** (foreign keys, unique, check)
- ✅ **All indexes** for performance
- ✅ **Row-level security** policies
- ✅ **Business logic functions** (credit deduction, slug generation)
- ✅ **Triggers** for automation
- ✅ **Pricing structure** (4 plans: Free, Pro, Agency, Enterprise)

### **What This Document Does NOT Contain:**
- ❌ Executable SQL (that comes after approval)
- ❌ Cloudflare Worker code
- ❌ Frontend implementation details

### **Tech Stack:**
- **Database:** Supabase (PostgreSQL 15)
- **Backend:** Cloudflare Workers (Hono framework)
- **Payments:** Stripe
- **AI Providers:** OpenAI, Anthropic, Apify
- **Secrets:** AWS Secrets Manager (ALL secrets stored here, not Cloudflare env vars)

---

# 📋 PART 1: PRICING & PLANS

## **FINAL PRICING STRUCTURE** (Overwrites All Previous Mentions)

| Plan ID | Name | Monthly Price | Credits/Month | Max Businesses | Stripe Product Needed? |
|---------|------|---------------|---------------|----------------|------------------------|
| `free` | Free Plan | $0 | 25 | 1 | No (customer only) |
| `pro` | Pro Plan | $30 | 100 | 3 | Yes |
| `agency` | Agency Plan | $80 | 300 | 10 | Yes |
| `enterprise` | Enterprise Plan | $300 | 1500 | Unlimited | Yes |

### **Credit Consumption:**
- **Light Analysis:** 1 credit
- **Deep Analysis:** 2 credits

### **Effective Analysis Capacity:**
- **Free:** 12 deep OR 25 light analyses/month
- **Pro:** 50 deep OR 100 light analyses/month
- **Agency:** 150 deep OR 300 light analyses/month
- **Enterprise:** 750 deep OR 1500 light analyses/month

### **Credit Rollover:**
- ✅ Credits **accumulate** (do not reset monthly)
- ✅ Industry standard behavior (AWS, Stripe, etc.)
- ⚠️ Can add cap later (e.g., max 3 months worth) if needed

### **Plans Table Features JSON:**
```json
{
  "max_businesses": 1  // Only this field
}
// NO "max_analyses_per_month" (redundant with credits_per_month)
```

---

# 📋 PART 2: COMPLETE TABLE STRUCTURES

## **Database Layers Overview:**

```
LAYER 1: Identity & Auth (1 table)
  └── users

LAYER 2: Billing Foundation (10 tables)
  ├── accounts
  ├── account_members
  ├── plans
  ├── subscriptions
  ├── credit_ledger
  ├── credit_balances (materialized view)
  ├── account_usage_summary
  ├── platform_metrics_daily
  ├── stripe_invoices
  └── webhook_events

LAYER 3: Business Logic (4 tables)
  ├── business_profiles
  ├── leads
  ├── analyses
  └── ai_usage_logs
```

**Total: 15 tables**

---

## LAYER 1: IDENTITY & AUTHENTICATION

### **Table: users**
**Purpose:** Core user identity (extends Supabase auth.users)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY, FK → auth.users(id) ON DELETE CASCADE | Supabase auth user ID |
| email | text | UNIQUE, NOT NULL | User email address |
| full_name | text | - | Display name (e.g., "Hamza Williams") |
| signature_name | text | - | Name used in AI-generated messages |
| avatar_url | text | - | Profile picture URL |
| onboarding_completed | boolean | DEFAULT false | Has user completed onboarding? |
| is_admin | boolean | DEFAULT false | Global admin (bypasses ALL RLS) |
| is_suspended | boolean | DEFAULT false | User suspended by admin |
| suspended_at | timestamptz | - | When suspension occurred |
| suspended_reason | text | - | Admin notes on why suspended |
| last_seen_at | timestamptz | - | Last activity timestamp |
| created_at | timestamptz | DEFAULT NOW() | Account creation time |
| updated_at | timestamptz | DEFAULT NOW() | Last update time (auto-trigger) |

**Indexes:**
- `idx_users_email` on `email`
- `idx_users_admin` on `is_admin` WHERE `is_admin = true`

**RLS:** No RLS (auth.users table handles this)

---

## LAYER 2: BILLING FOUNDATION

### **Table: accounts**
**Purpose:** The billable entity (NOT users). One account = one Stripe customer.

**Key Concept:** Users can belong to multiple accounts, but credits belong to ACCOUNTS.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY, DEFAULT gen_random_uuid() | Account unique identifier |
| owner_id | uuid | NOT NULL, FK → users(id) ON DELETE RESTRICT | Account owner (can delete account) |
| name | text | NOT NULL | Display name (e.g., "Hamza's Account") |
| slug | text | UNIQUE | URL-safe identifier (e.g., "hamza-williams") |
| is_suspended | boolean | DEFAULT false | Account suspended by admin |
| suspended_at | timestamptz | - | When suspension occurred |
| suspended_reason | text | - | Admin notes |
| suspended_by | uuid | FK → users(id) | Which admin suspended |
| deleted_at | timestamptz | - | Soft delete timestamp |
| created_at | timestamptz | DEFAULT NOW() | - |
| updated_at | timestamptz | DEFAULT NOW() | Auto-updated via trigger |

**Indexes:**
- `idx_accounts_owner` on `owner_id`
- `idx_accounts_slug` on `slug` WHERE `deleted_at IS NULL`
- `idx_accounts_active` on `id` WHERE `deleted_at IS NULL AND is_suspended = false`

**Constraints:**
- `slug` must be unique
- `owner_id` cannot be deleted (RESTRICT)

**RLS:** Users see only accounts they belong to (via account_members)

**Naming Pattern:**
- On signup: `{full_name}'s Account` (e.g., "Hamza Williams's Account")
- Slug generation: Function `generate_slug(full_name)` → "hamza-williams"

---

### **Table: account_members**
**Purpose:** Junction table for multi-user accounts (team collaboration)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY | - |
| account_id | uuid | NOT NULL, FK → accounts(id) ON DELETE CASCADE | Which account |
| user_id | uuid | NOT NULL, FK → users(id) ON DELETE CASCADE | Which user |
| role | text | DEFAULT 'member', CHECK | User's role in account |
| invited_by | uuid | FK → users(id) | Who invited this user |
| joined_at | timestamptz | DEFAULT NOW() | When they joined |
| created_at | timestamptz | DEFAULT NOW() | - |

**Constraints:**
- `UNIQUE (account_id, user_id)` - no duplicate memberships
- `CHECK (role IN ('owner', 'admin', 'member', 'viewer'))` - valid roles only

**Role Definitions:**
- **owner:** Full control, can delete account, manage all members
- **admin:** Can manage members/viewers (but not other admins)
- **member:** Can spend credits, create leads/analyses
- **viewer:** Read-only access

**Indexes:**
- `idx_account_members_user` on `user_id, account_id`
- `idx_account_members_account` on `account_id`

**RLS:** 
- Phase 1: All members have equal access (role enforcement deferred)
- Phase 2: Role-based permissions (owner/admin can manage, member can execute, viewer read-only)

---

### **Table: plans**
**Purpose:** Define available subscription tiers (data-driven, not hardcoded)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | text | PRIMARY KEY | Plan identifier ('free', 'pro', 'agency', 'enterprise') |
| name | text | NOT NULL | Display name ("Free Plan", "Pro Plan") |
| credits_per_month | integer | NOT NULL, CHECK >= 0 | Monthly credit grant (25, 100, 300, 1500) |
| price_cents | integer | NOT NULL, CHECK >= 0 | Monthly price in cents (0, 3000, 8000, 30000) |
| stripe_price_id | text | - | Stripe Price ID (NULL for free, filled after Stripe setup) |
| is_active | boolean | DEFAULT true | Can new users sign up for this plan? |
| features | jsonb | - | `{"max_businesses": 1}` |
| created_at | timestamptz | DEFAULT NOW() | - |

**Preseed Data:**
```
free     | Free Plan       | 25   | 0     | NULL | {"max_businesses": 1}
pro      | Pro Plan        | 100  | 3000  | NULL | {"max_businesses": 3}
agency   | Agency Plan     | 300  | 8000  | NULL | {"max_businesses": 10}
enterprise | Enterprise  | 1500 | 30000 | NULL | {"max_businesses": null}
```

**Notes:**
- `stripe_price_id` filled later (after creating products in Stripe dashboard)
- `features.max_businesses` = null means unlimited
- Plans are EDITABLE (can change credits/price anytime)
- Changing `credits_per_month` affects NEXT month's renewal
- Changing `price_cents` requires NEW Stripe price (old subscriptions unchanged)

**RLS:** Public read access (for pricing page)

---

### **Table: subscriptions**
**Purpose:** Track subscription history (append-only, never UPDATE plan_type)

**CRITICAL RULE:** NEVER mutate existing subscriptions. Always INSERT new row when upgrading/downgrading.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY | - |
| account_id | uuid | NOT NULL, FK → accounts(id) ON DELETE RESTRICT | Which account |
| plan_type | text | NOT NULL, FK → plans(id) | Plan ID ('free', 'pro', etc.) |
| price_cents | integer | NOT NULL, CHECK >= 0 | Price at time of subscription |
| stripe_customer_id | text | - | Stripe customer ID ("cus_...") |
| stripe_subscription_id | text | - | Stripe subscription ID ("sub_...") - NULL for free |
| stripe_price_id | text | - | Stripe price ID used |
| status | text | DEFAULT 'active', CHECK | Subscription state |
| current_period_start | timestamptz | NOT NULL | Billing period start |
| current_period_end | timestamptz | NOT NULL | Billing period end |
| canceled_at | timestamptz | - | When user canceled (ends at period_end) |
| created_at | timestamptz | DEFAULT NOW() | - |

**Valid Statuses:**
- `active` - Currently active, credits granted monthly
- `canceled` - Canceled, still active until period_end
- `past_due` - Payment failed (Stripe webhook)
- `paused` - Admin paused (no credit grants)
- `pending_stripe_confirmation` - Just signed up, waiting for Stripe webhook

**Constraints:**
- Only ONE active subscription per account: `UNIQUE INDEX idx_one_active_subscription ON subscriptions(account_id) WHERE status = 'active'`
- `CHECK (status IN (...))`

**Indexes:**
- `idx_subscriptions_active` on `account_id, status` WHERE `status = 'active'`
- `idx_subscriptions_customer` on `stripe_customer_id`

**Upgrade/Downgrade Flow:**
```
User upgrades from Free to Pro:
1. UPDATE old_sub SET status='canceled', canceled_at=NOW() WHERE account_id=X AND status='active'
2. INSERT new_sub (account_id=X, plan_type='pro', status='active', ...)
Result: Full history preserved (free → pro audit trail)
```

**Free Plan Subscription:**
- Has `stripe_customer_id` (Stripe customer created)
- Has NO `stripe_subscription_id` (no Stripe subscription object)
- Status is immediately `active` (no payment needed)
- Still has periods (current_period_start/end for monthly renewal)

**RLS:** Users can read their own subscriptions (read-only)

---

### **Table: credit_ledger**
**Purpose:** Append-only financial audit log (NEVER UPDATE, only INSERT)

**This is the source of truth for all credit movements.**

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY | - |
| account_id | uuid | NOT NULL, FK → accounts(id) ON DELETE RESTRICT | Which account |
| amount | integer | NOT NULL | Credits added/removed (+100, -2) |
| balance_after | integer | NOT NULL, CHECK >= 0 | Balance after this transaction |
| transaction_type | text | NOT NULL, CHECK | Type of transaction |
| reference_type | text | - | What caused this? ('analysis', 'subscription', etc.) |
| reference_id | uuid | - | ID of the related record (analysis_id, subscription_id) |
| description | text | - | Human-readable description |
| metadata | jsonb | - | Additional context (analysis details, etc.) |
| created_by | uuid | FK → users(id) | Who triggered (NULL for system) |
| created_at | timestamptz | DEFAULT NOW() | Transaction timestamp |

**Valid Transaction Types:**
- `subscription_renewal` - Monthly credit grant
- `analysis` - Credit deduction for analysis
- `refund` - Manual refund by admin
- `admin_grant` - Admin manually added credits
- `chargeback` - Stripe chargeback (negative)
- `signup_bonus` - Welcome credits (25 free on signup)

**Constraints:**
- `CHECK (balance_after >= 0)` - Cannot overdraft
- `CHECK (transaction_type IN (...))`

**Indexes:**
- `idx_credit_ledger_account` on `account_id, created_at DESC`
- `idx_credit_ledger_type` on `transaction_type, created_at DESC`

**Example Rows:**
```
| account_id | amount | balance_after | type                | description              |
|------------|--------|---------------|---------------------|--------------------------|
| acc_123    | +25    | 25            | signup_bonus        | Welcome to Oslira!       |
| acc_123    | -2     | 23            | analysis            | Deep analysis of @nike   |
| acc_123    | +100   | 123           | subscription_renewal| Monthly Pro renewal      |
```

**Retention:** Forever (legal/accounting requirement)

**RLS:** Users can read their own ledger (read-only, no inserts allowed)

---

### **Table: credit_balances**
**Purpose:** Materialized view of current credit balance (performance optimization)

**Why Needed:** Querying `SUM(amount) FROM credit_ledger` is slow at scale. This table caches the result.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| account_id | uuid | PRIMARY KEY, FK → accounts(id) ON DELETE CASCADE | Which account |
| current_balance | integer | NOT NULL, DEFAULT 0, CHECK >= 0 | Current credits |
| last_transaction_at | timestamptz | - | Last credit_ledger entry |
| created_at | timestamptz | DEFAULT NOW() | - |
| updated_at | timestamptz | DEFAULT NOW() | Auto-updated via trigger |

**How It Works:**
1. User calls `deduct_credits()` function
2. Function inserts into `credit_ledger`
3. Trigger `credit_ledger_update_balance` fires
4. Trigger updates `credit_balances.current_balance`
5. Dashboard queries `credit_balances` (instant!)

**Indexes:**
- `idx_credit_balances_account` on `account_id`

**RLS:** Users can read their own balance (read-only)

---

### **Table: account_usage_summary**
**Purpose:** Monthly rollup of account activity (performance optimization for dashboards)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY | - |
| account_id | uuid | NOT NULL, FK → accounts(id) ON DELETE RESTRICT | Which account |
| period_start | date | NOT NULL | Month start (2025-01-01) |
| period_end | date | NOT NULL | Month end (2025-01-31) |
| credits_used | integer | DEFAULT 0 | Total credits spent this month |
| light_analyses_count | integer | DEFAULT 0 | Number of light analyses |
| deep_analyses_count | integer | DEFAULT 0 | Number of deep analyses |
| total_analyses_count | integer | DEFAULT 0 | light + deep |
| total_cost_cents | integer | DEFAULT 0 | Actual AI API costs (for margin calculation) |
| is_finalized | boolean | DEFAULT false | Month ended? (frozen on 1st of next month) |
| created_at | timestamptz | DEFAULT NOW() | - |
| updated_at | timestamptz | DEFAULT NOW() | Auto-updated during month |

**Constraints:**
- `UNIQUE (account_id, period_start)` - one summary per account per month

**Indexes:**
- `idx_account_usage` on `account_id, period_start DESC`

**Update Pattern:**
- **During month:** Updated real-time (on each analysis completion)
- **End of month:** Cron sets `is_finalized = true` (freezes data)
- **Next month:** New row created

**RLS:** Users can read their own summaries (read-only)

---

### **Table: platform_metrics_daily**
**Purpose:** Daily rollup of platform-wide statistics (admin dashboard, no RLS)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY | - |
| metric_date | date | NOT NULL, UNIQUE | Date of metrics (2025-01-15) |
| total_analyses_count | integer | DEFAULT 0 | Total analyses run |
| light_analyses_count | integer | DEFAULT 0 | Light analyses |
| deep_analyses_count | integer | DEFAULT 0 | Deep analyses |
| active_accounts_count | integer | DEFAULT 0 | Accounts with activity today |
| new_accounts_count | integer | DEFAULT 0 | Signups today |
| openai_calls_count | integer | DEFAULT 0 | OpenAI API calls |
| anthropic_calls_count | integer | DEFAULT 0 | Anthropic API calls |
| apify_calls_count | integer | DEFAULT 0 | Apify scraper calls |
| openai_cost_cents | integer | DEFAULT 0 | OpenAI costs |
| anthropic_cost_cents | integer | DEFAULT 0 | Anthropic costs |
| apify_cost_cents | integer | DEFAULT 0 | Apify costs |
| total_cost_cents | integer | DEFAULT 0 | Total AI costs |
| openai_avg_duration_ms | integer | DEFAULT 0 | Avg response time |
| anthropic_avg_duration_ms | integer | DEFAULT 0 | Avg response time |
| apify_avg_duration_ms | integer | DEFAULT 0 | Avg scrape time |
| openai_error_count | integer | DEFAULT 0 | Failed calls |
| anthropic_error_count | integer | DEFAULT 0 | Failed calls |
| apify_error_count | integer | DEFAULT 0 | Failed scrapes |
| credits_purchased_count | integer | DEFAULT 0 | Total credits granted (renewals + purchases) |
| revenue_cents | integer | DEFAULT 0 | Stripe payments received |
| profit_margin_cents | integer | DEFAULT 0 | Revenue - AI costs |
| daily_active_users | integer | DEFAULT 0 | Unique users active |
| avg_lead_score | numeric | - | Average analysis score |
| high_quality_leads_count | integer | DEFAULT 0 | Leads with score > 80 |
| mrr_cents | integer | DEFAULT 0 | Monthly Recurring Revenue |
| arr_cents | integer | DEFAULT 0 | Annual Recurring Revenue |
| customer_count | integer | DEFAULT 0 | Total paying customers |
| created_at | timestamptz | DEFAULT NOW() | - |
| updated_at | timestamptz | DEFAULT NOW() | - |

**Indexes:**
- `idx_platform_metrics_date` on `metric_date DESC`

**Retention:** Forever (historical analytics)

**RLS:** No RLS (service role only, admins query via backend API)

---

### **Table: stripe_invoices**
**Purpose:** Minimal invoice tracking (support tickets, revenue dashboard)

**Why Needed:**
- Support can quickly see payment history without Stripe API call
- Revenue dashboard doesn't need Stripe exports
- Webhook durability (record persists even if processing fails)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY | - |
| account_id | uuid | NOT NULL, FK → accounts(id) ON DELETE RESTRICT | Which account |
| stripe_invoice_id | text | UNIQUE, NOT NULL | Stripe invoice ID ("in_...") |
| stripe_subscription_id | text | - | Related subscription |
| amount_cents | integer | NOT NULL | Invoice amount |
| currency | text | DEFAULT 'usd' | Currency code |
| status | text | NOT NULL, CHECK | Invoice status |
| paid_at | timestamptz | - | When payment succeeded |
| failed_at | timestamptz | - | When payment failed |
| failure_reason | text | - | Why payment failed |
| created_at | timestamptz | DEFAULT NOW() | - |

**Valid Statuses:**
- `paid` - Successfully paid
- `open` - Awaiting payment
- `void` - Canceled/voided
- `uncollectible` - Payment failed, given up

**Constraints:**
- `CHECK (status IN ('paid', 'open', 'void', 'uncollectible'))`

**Indexes:**
- `idx_invoices_account` on `account_id, created_at DESC`
- `idx_invoices_status` on `status, paid_at DESC`
- `idx_invoices_stripe_id` on `stripe_invoice_id`

**Populated By:** Stripe webhooks (`invoice.paid`, `invoice.payment_failed`)

**RLS:** Users can read their own invoices (read-only)

---

### **Table: webhook_events**
**Purpose:** Idempotency tracking for Stripe webhooks (prevent duplicate processing)

**Why Critical:** Stripe retries webhooks on timeout. Without deduplication, user gets double credits.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| stripe_event_id | text | PRIMARY KEY | Stripe event ID ("evt_...") |
| event_type | text | NOT NULL | Event type ("invoice.paid") |
| account_id | uuid | FK → accounts(id) | Related account (if identifiable) |
| processed_at | timestamptz | DEFAULT NOW() | When we processed it |
| payload | jsonb | - | Full Stripe event JSON (debugging) |

**Webhook Handler Pattern:**
```
1. Webhook arrives with event_id = "evt_123"
2. Check: SELECT 1 FROM webhook_events WHERE stripe_event_id = 'evt_123'
3. If exists: return 200 OK (already processed)
4. Process event (grant credits, update subscription, etc.)
5. INSERT INTO webhook_events (stripe_event_id, ...)
6. Return 200 OK
```

**Indexes:**
- `idx_webhook_stripe_id` on `stripe_event_id` (UNIQUE, critical for dedup)
- `idx_webhook_type` on `event_type, processed_at DESC`

**Retention:** Forever (audit trail)

**RLS:** No RLS (service role only)

---

## LAYER 3: BUSINESS LOGIC

### **Table: business_profiles**
**Purpose:** User's business context (used for AI analysis prompts)

**Key Design:** Minimal direct columns, most data in JSONB (flexible, rarely edited)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY | - |
| account_id | uuid | NOT NULL, FK → accounts(id) ON DELETE RESTRICT | Which account owns this |
| business_name | text | NOT NULL | Display name (e.g., "Oslira Marketing") |
| website | text | - | Business website URL |
| business_one_liner | text | - | Short tagline (dashboard greeting) |
| business_context_pack | jsonb | - | AI-generated context (niche, audience, value prop, etc.) |
| context_version | text | DEFAULT 'v1.0' | AI prompt version used |
| context_generated_at | timestamptz | - | When AI generated this |
| context_manually_edited | boolean | DEFAULT false | Did user edit manually? |
| context_updated_at | timestamptz | - | Last manual edit |
| deleted_at | timestamptz | - | Soft delete timestamp |
| created_at | timestamptz | DEFAULT NOW() | - |
| updated_at | timestamptz | DEFAULT NOW() | Auto-updated via trigger |

**Business Context Pack Structure (JSONB):**
```json
{
  "niche": "B2B SaaS copywriting",
  "target_audience": "Tech founders, marketing managers",
  "value_proposition": "Conversion-focused copy that sells",
  "problems_solved": ["Low conversion rates", "Generic messaging"],
  "unique_approach": "Data-driven storytelling",
  "ideal_client_traits": ["Fast-growing startups", "Tech-savvy"],
  "service_offerings": ["Landing pages", "Email sequences"],
  "tone_of_voice": "Professional yet approachable",
  "competitors": ["Agency X", "Freelancer Y"],
  "geographic_focus": "North America"
}
```

**Settings UI Pattern:**
```
┌─ Business Profile ────────────────┐
│ Name: [Oslira Marketing        ]  │ ← Direct edit
│ Website: [oslira.com           ]  │ ← Direct edit
│ Tagline: [We write copy that sells]│ ← Direct edit
│                                   │
│ [Regenerate Business Context]  ← Opens onboarding form,
│                                   recreates business_context_pack
└───────────────────────────────────┘
```

**Indexes:**
- `idx_business_profiles_account` on `account_id, created_at DESC` WHERE `deleted_at IS NULL`

**Constraints:**
- Account can have multiple business profiles (agency use case)

**Soft Delete:** Yes (30-day recovery window)

**RLS:** Users see profiles from their accounts only

---

### **Table: leads**
**Purpose:** Instagram accounts being analyzed (CRM records)

**Key Design:** Profile data is PATCHED on each re-analysis (follower count updates, bio changes, etc.)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY | - |
| account_id | uuid | NOT NULL, FK → accounts(id) ON DELETE RESTRICT | Which account owns this lead |
| business_profile_id | uuid | FK → business_profiles(id) ON DELETE SET NULL | Which business context used |
| instagram_username | text | NOT NULL | Instagram handle (e.g., "nike") |
| display_name | text | - | Display name from profile |
| profile_pic_url | text | - | Profile picture URL |
| profile_url | text | - | Full Instagram URL |
| follower_count | integer | - | Followers (updated on re-analysis) |
| following_count | integer | - | Following count |
| post_count | integer | - | Total posts |
| bio | text | - | Profile bio text |
| external_url | text | - | Link in bio |
| is_verified | boolean | DEFAULT false | Blue checkmark? |
| is_private | boolean | DEFAULT false | Private account? |
| is_business_account | boolean | DEFAULT false | Business/Creator account? |
| platform | text | DEFAULT 'instagram' | Future: TikTok, YouTube, etc. |
| first_analyzed_at | timestamptz | - | First time analyzed |
| last_analyzed_at | timestamptz | - | Most recent analysis (updated on re-analysis) |
| deleted_at | timestamptz | - | Soft delete |
| created_at | timestamptz | DEFAULT NOW() | - |

**CRITICAL CONSTRAINT:**
- `UNIQUE (account_id, business_profile_id, instagram_username)`
- **Allows:** Same Instagram account analyzed for DIFFERENT businesses
- **Example:** Agency analyzes @nike for "Campaign A" AND "Campaign B" (2 separate lead records)

**Indexes:**
- `idx_leads_active` on `account_id, last_analyzed_at DESC` WHERE `deleted_at IS NULL`
- `idx_leads_username_lookup` on `instagram_username` WHERE `deleted_at IS NULL`
- `idx_leads_business` on `business_profile_id, created_at DESC`

**Soft Delete:** Yes (30-day recovery)

**RLS:** Users see leads from their accounts only

---

### **Table: analyses**
**Purpose:** AI analysis results (supports re-analysis, historical tracking)

**Key Design:** Multiple analyses allowed per lead (track growth over time)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY | - |
| lead_id | uuid | NOT NULL, FK → leads(id) ON DELETE CASCADE | Which lead was analyzed |
| account_id | uuid | NOT NULL, FK → accounts(id) ON DELETE RESTRICT | Which account (for RLS) |
| business_profile_id | uuid | FK → business_profiles(id) ON DELETE SET NULL | Business context used |
| requested_by | uuid | FK → users(id) ON DELETE SET NULL | Which user triggered |
| analysis_type | text | NOT NULL, CHECK | 'light' or 'deep' |
| analysis_version | text | - | AI prompt version (e.g., "v2.3") |
| status | text | DEFAULT 'pending', CHECK | Analysis lifecycle |
| ai_response | jsonb | - | Full AI response stored inline |
| overall_score | integer | - | Extracted from AI response (0-100) |
| niche_fit_score | integer | - | Extracted score |
| engagement_score | integer | - | Extracted score |
| confidence_level | numeric | - | AI confidence (0.0-1.0) |
| credits_charged | integer | - | Credits deducted (1 or 2) |
| total_cost_cents | integer | - | Actual API costs (for margin calc) |
| model_used | text | - | Which AI model ("gpt-4", "claude-3.5-sonnet") |
| processing_duration_ms | integer | - | Total time to complete |
| started_at | timestamptz | - | When analysis began |
| completed_at | timestamptz | - | When analysis finished |
| deleted_at | timestamptz | - | Soft delete |
| created_at | timestamptz | DEFAULT NOW() | - |

**Valid Analysis Types:**
- `light` - Quick analysis (1 credit)
- `deep` - Comprehensive analysis (2 credits)

**Valid Statuses:**
- `pending` - Queued
- `processing` - AI is working
- `completed` - Finished successfully
- `failed` - Error occurred

**Constraints:**
- `CHECK (analysis_type IN ('light', 'deep'))`
- `CHECK (status IN ('pending', 'processing', 'completed', 'failed'))`

**AI Response Structure (JSONB):**
```json
{
  "overall_score": 85,
  "niche_fit_score": 90,
  "engagement_score": 80,
  "confidence_level": 0.92,
  "summary": "Excellent fit for B2B SaaS outreach...",
  "strengths": ["High engagement", "Relevant niche"],
  "concerns": ["Audience size", "Recent activity"],
  "outreach_suggestions": ["Personalized DM", "Comment on recent post"],
  "message_template": "Hi [name], I noticed..."
}
```

**Indexes:**
- `idx_analyses_latest` on `lead_id, completed_at DESC` WHERE `deleted_at IS NULL`
- `idx_analyses_in_progress` on `lead_id, status` WHERE `status IN ('pending', 'processing')`
- `idx_analyses_requester` on `requested_by, created_at DESC`

**Deduplication Logic (Application-Level):**
```
Before creating new analysis:
1. Check: SELECT * FROM analyses WHERE lead_id=X AND status IN ('pending','processing')
2. If exists: Return error "Analysis already in progress"
3. Else: Proceed
```

**Re-Analysis Allowed:** Yes (no constraint on analyses.lead_id)

**Soft Delete:** Yes (30-day recovery)

**RLS:** Users see analyses from their accounts only

---

### **Table: ai_usage_logs**
**Purpose:** Track every AI API call (performance monitoring, cost analysis)

**CRITICAL SEPARATION:** This is NOT for billing. User sees charges in `credit_ledger`. This is for PLATFORM METRICS.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PRIMARY KEY | - |
| account_id | uuid | NOT NULL, FK → accounts(id) ON DELETE RESTRICT | Which account (for cost attribution) |
| analysis_id | uuid | FK → analyses(id) ON DELETE SET NULL | Related analysis (if any) |
| provider | text | NOT NULL, CHECK | 'openai', 'anthropic', 'apify' |
| model | text | - | Specific model ("gpt-4-turbo", "claude-3.5-sonnet") |
| api_call_type | text | - | 'scrape', 'analysis', 'message_generation' |
| tokens_input | integer | - | Input tokens (OpenAI/Anthropic) |
| tokens_output | integer | - | Output tokens |
| cost_cents | integer | NOT NULL, CHECK >= 0 | Actual API cost |
| duration_ms | integer | - | Response time |
| status | text | CHECK | 'success', 'error', 'timeout' |
| error_message | text | - | Error details (if failed) |
| started_at | timestamptz | - | API call start |
| completed_at | timestamptz | - | API call end |
| created_at | timestamptz | DEFAULT NOW() | - |

**Constraints:**
- `CHECK (provider IN ('openai', 'anthropic', 'apify'))`
- `CHECK (status IN ('success', 'error', 'timeout'))`
- `CHECK (cost_cents >= 0)`

**Example Flow (1 Deep Analysis = 3 AI Usage Logs):**
```
1. Apify scrape Instagram profile
   → INSERT ai_usage_logs (provider='apify', cost_cents=5, duration_ms=4500)

2. OpenAI analyzes scraped data
   → INSERT ai_usage_logs (provider='openai', model='gpt-4', cost_cents=12, tokens_input=2500)

3. Claude generates outreach message
   → INSERT ai_usage_logs (provider='anthropic', model='claude-3.5-sonnet', cost_cents=8)

Total: 3 logs, total_cost_cents = 25¢
User charged: 2 credits (not 25¢ - different systems!)
```

**Indexes:**
- `idx_ai_usage_account` on `account_id, created_at DESC`
- `idx_ai_usage_provider` on `provider, created_at DESC`
- `idx_ai_usage_analysis` on `analysis_id`

**Retention:** 90 days only (purged by daily cron)

**RLS:** No RLS (service role only, aggregated in platform_metrics_daily)

---

# 📋 PART 3: ARCHITECTURAL DECISIONS

## **1. BILLING MODEL: Account-Based (Not User-Based)**

### **Hierarchy:**
```
users (identity)
  └── account_members (junction)
      └── accounts (THE BILLABLE ENTITY)
          ├── subscriptions (Stripe lives here)
          ├── credit_ledger (credit movements)
          ├── credit_balances (materialized balance)
          └── business_profiles (shared credits across businesses)
              └── leads & analyses
```

### **Why Account-Based:**
- ✅ One user can own 5 businesses under ONE account with SHARED credit pool
- ✅ Enables team collaboration (multiple users, one account, shared credits)
- ✅ Industry standard (Notion, Slack, Figma, Stripe all use this)
- ✅ Easier to add team members later without schema changes

### **Example Scenario:**
```
User: hamza@example.com
  └── Account: "Hamza's Agency" (100 credits)
      ├── Business Profile: "Client A - Fashion Brand"
      ├── Business Profile: "Client B - Tech Startup"
      └── Business Profile: "Client C - Food Blog"

All 3 businesses share the same 100 credit pool.
```

---

## **2. CREDIT SYSTEM: Dual-Table Architecture**

### **Two Tables with Different Purposes:**

#### **A) credit_ledger (Append-Only Audit Log)**
- **Purpose:** Financial audit trail, NEVER UPDATE only INSERT
- **Stores:** Every credit movement (+100 renewal, -2 analysis)
- **Source of Truth:** Current balance = last `balance_after` value
- **Retention:** Forever (legal/accounting requirement)
- **Query Speed:** Slow at scale (millions of rows)

#### **B) credit_balances (Materialized View)**
- **Purpose:** Performance optimization for dashboard queries
- **Stores:** Current balance (cached from ledger)
- **Updated:** Automatically via trigger on every ledger INSERT
- **Retention:** Forever
- **Query Speed:** Instant (single row lookup)

### **Why Both:**
- `credit_ledger` = trustworthy audit (never delete, never mutate)
- `credit_balances` = fast queries (don't scan millions of rows)

### **Flow Example:**
```
1. User triggers deep analysis
2. Cloudflare Worker calls: deduct_credits(account_id, -2, 'analysis', ...)
3. Function locks credit_balances row (prevents race conditions)
4. Function checks: balance >= 2? (prevent overdraft)
5. Function INSERTs into credit_ledger: amount=-2, balance_after=98
6. Trigger fires: UPDATE credit_balances SET current_balance=98
7. Analysis proceeds
```

---

## **3. AI COST TRACKING: Separate from Credit Accounting**

### **CRITICAL SEPARATION:** Billing ≠ Performance Monitoring

#### **A) credit_ledger (What User Paid)**
- Tracks: User charged 2 credits
- Does NOT store: OpenAI costs, token counts, provider details

#### **B) ai_usage_logs (Actual API Costs)**
- Tracks: Every OpenAI/Claude/Apify call separately
- 1 analysis = 3-8 rows (scrape + AI analysis + message generation)
- Stores: provider, model, tokens, cost_cents, duration_ms
- **Retention:** 90 days only (archived after)

#### **C) platform_metrics_daily (Admin Dashboard)**
- Daily rollup of platform-wide stats
- Aggregates from `ai_usage_logs` + other sources
- Fast admin queries without scanning millions of rows

### **Flow Example:**
```
User runs deep analysis:
1. Apify scrape → INSERT ai_usage_logs (cost: 5¢, duration: 4.5s)
2. OpenAI analysis → INSERT ai_usage_logs (cost: 12¢, tokens: 2500)
3. Claude message → INSERT ai_usage_logs (cost: 8¢, tokens: 1800)
4. Total cost: 25¢
5. Charge user: INSERT credit_ledger (amount: -2, metadata: {actual_cost_cents: 25})
6. Update rollups: account_usage_summary, platform_metrics_daily
```

### **Why Separate:**
- User doesn't care about token counts (just credit cost)
- Platform needs AI metrics for: profit margin, provider comparison, performance tracking
- Different retention policies (ledger forever, logs 90 days)

---

## **4. SUBSCRIPTIONS: Append-Only (Never UPDATE plan_type)**

### **Rule:** NEVER mutate existing subscription rows

```
❌ WRONG:
UPDATE subscriptions SET plan_type = 'pro' WHERE id = 'xxx';

✅ CORRECT:
-- 1. Cancel old subscription
UPDATE subscriptions 
SET status = 'canceled', canceled_at = NOW() 
WHERE account_id = 'acc_123' AND status = 'active';

-- 2. Insert new subscription
INSERT INTO subscriptions (account_id, plan_type, status, ...) 
VALUES ('acc_123', 'pro', 'active', ...);
```

### **Why:**
- ✅ Preserves full subscription history (free → pro → enterprise audit trail)
- ✅ Supports refunds (know what user paid at specific time)
- ✅ Supports proration (calculate based on historical prices)
- ✅ Legal compliance (financial audit trail)

---

## **5. STRIPE INTEGRATION: Optimistic Credit Grant**

### **Flow:**
```
1. User signs up → Supabase auth.users created
   ↓
2. IMMEDIATE (optimistic, no waiting):
   - INSERT accounts
   - INSERT subscriptions (status = 'pending_stripe_confirmation')
   - INSERT credit_ledger (+25 signup bonus)
   - User can START using app immediately (analyze leads!)
   ↓
3. BACKGROUND (async, 2-5 seconds):
   - Cloudflare Worker calls Stripe API
   - Creates customer: stripe.customers.create()
   - For free: No subscription object (just customer)
   - For paid: Create subscription: stripe.subscriptions.create()
   ↓
4. WEBHOOK (when Stripe confirms):
   - Receive: customer.subscription.created (or invoice.paid)
   - UPDATE subscriptions SET 
       stripe_customer_id = 'cus_...', 
       stripe_subscription_id = 'sub_...',
       status = 'active'
   ↓
5. IF STRIPE FAILS (network error, etc.):
   - Cron job retries every 5 minutes
   - User KEEPS their 25 credits (optimistic grant stands)
   - Eventually succeeds or admin investigates
```

### **Critical Fields:**
```sql
subscriptions (
  stripe_customer_id text,  -- ← TEXT not UUID (Stripe returns "cus_abc123")
  stripe_subscription_id text,  -- ← TEXT not UUID (returns "sub_xyz789")
  status text  -- 'pending_stripe_confirmation' → 'active'
)
```

### **Free Plan Special Case:**
- Has `stripe_customer_id` (customer always created)
- Has NO `stripe_subscription_id` (NULL - no Stripe subscription object)
- Status is immediately `active` (no payment needed)
- Still has periods (current_period_start/end for monthly cron renewal)

---

## **6. WEBHOOK IDEMPOTENCY: Prevent Duplicate Processing**

### **Problem:** 
Stripe retries webhooks on timeout. Without deduplication, user gets double credits.

### **Solution: webhook_events Table**

```sql
CREATE TABLE webhook_events (
  stripe_event_id text PRIMARY KEY,  -- "evt_1A2B3C..."
  event_type text NOT NULL,
  account_id uuid,
  processed_at timestamptz DEFAULT NOW(),
  payload jsonb
)
```

### **Handler Pattern:**
```javascript
// ALWAYS check FIRST before processing
const existing = await db.query(
  'SELECT 1 FROM webhook_events WHERE stripe_event_id = $1',
  [event.id]
);

if (existing.rows.length > 0) {
  return { received: true, message: 'already_processed' };
}

// Process event (grant credits, update subscription, etc.)
// ...

// Then record it
await db.insert('webhook_events', {
  stripe_event_id: event.id,
  event_type: event.type,
  account_id: accountId,
  payload: event
});
```

### **Why Critical:**
- Stripe sends webhook
- Your worker processes (grants 100 credits)
- Network timeout before response
- Stripe retries webhook
- Without dedup: User gets 200 credits (BAD!)
- With dedup: Worker sees existing event_id, skips processing (GOOD!)

---

## **7. LEADS + ANALYSES: Separate Tables, JSONB Inline**

### **Design Decision:** Keep separate (not denormalized)

```sql
leads (
  -- Profile data (PATCHED on each re-analysis)
  instagram_username text,
  follower_count integer,
  bio text,
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

### **Why No Denormalization:**
- ❌ Don't store `latest_score` in leads table (causes sync bugs)
- ✅ Use indexes + LATERAL joins for fast "latest analysis" queries
- ✅ Supports historical analysis tracking (UI feature for "track growth")
- ✅ Clean separation: leads = profile data, analyses = AI results

### **Query Pattern for "Latest Analysis":**
```sql
SELECT l.*, a.* 
FROM leads l
LEFT JOIN LATERAL (
  SELECT * FROM analyses 
  WHERE lead_id = l.id 
    AND deleted_at IS NULL
  ORDER BY completed_at DESC 
  LIMIT 1
) a ON true
WHERE l.account_id = $1 AND l.deleted_at IS NULL;
```

---

## **8. BUSINESS PROFILES: Minimal Schema, JSONB for Flexibility**

### **User-Editable Fields:**
```sql
business_profiles (
  business_name text,        -- Direct edit
  website text,              -- Direct edit
  business_one_liner text,   -- Direct edit
  
  -- Everything else in JSONB
  business_context_pack jsonb  -- {niche, audience, value_prop, ...}
)
```

### **Why JSONB:**
- ✅ Onboarding collects 15+ fields (niche, audience, problems, tone, etc.)
- ✅ AI generates `business_context_pack` from these during onboarding
- ✅ User rarely edits (set-and-forget)
- ✅ If editing needed: "Regenerate from Scratch" button shows onboarding form again
- ✅ Future-proof: Can add fields without ALTER TABLE

### **Settings UI Pattern:**
```
┌─ Business Profile ────────────────┐
│ Name: [Oslira Marketing        ]  │ ← Direct column edit
│ Website: [oslira.com           ]  │ ← Direct column edit
│ Tagline: [We write copy that sells]│ ← Direct column edit
│                                   │
│ Business Context (AI-generated):  │
│ ┌─────────────────────────────┐  │
│ │ Niche: B2B SaaS copywriting │  │
│ │ Audience: Tech founders     │  │
│ │ Value Prop: Conversion copy │  │
│ │ ... (10+ more fields)       │  │
│ └─────────────────────────────┘  │
│                                   │
│ [Regenerate Business Context]     │ ← Opens onboarding form
└───────────────────────────────────┘
```

---

## **9. SOFT DELETES: 30-Day Recovery Window**

### **Tables with deleted_at:**
- accounts
- business_profiles
- leads
- analyses

### **Query Pattern:**
```sql
-- ALWAYS filter out deleted records
SELECT * FROM leads 
WHERE account_id = $1 
  AND deleted_at IS NULL;
```

### **Cleanup Cron (Daily at 2 AM UTC):**
```sql
-- Hard delete after 30 days
DELETE FROM leads WHERE deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM analyses WHERE deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM business_profiles WHERE deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM accounts WHERE deleted_at < NOW() - INTERVAL '30 days';
```

### **Partial Indexes (Performance):**
```sql
-- Only index active records
CREATE INDEX idx_leads_active 
ON leads(account_id, last_analyzed_at DESC) 
WHERE deleted_at IS NULL;
```

### **User Experience:**
```
User deletes lead:
  → UPDATE leads SET deleted_at = NOW()
  → Analyses for that lead also soft-deleted (app logic, not CASCADE)
  → Lead disappears from dashboard
  → "Trash" view shows deleted leads (< 30 days old)
  → "Restore" button clears deleted_at
  
After 30 days:
  → Daily cron hard-deletes
  → Analyses CASCADE deleted (foreign key)
  → credit_ledger.lead_id → NULL (audit preserved)
  → Transaction shows: "2 credits charged for [deleted lead]"
```

---

## **10. FOREIGN KEY CASCADE RULES**

### **Strategy:** Preserve audit trail, cascade only where data is meaningless without parent

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

### **Example Flow:**
```
User soft-deletes lead:
  UPDATE leads SET deleted_at = NOW()
    ↓
  Analyses soft-deleted too (app logic, not CASCADE)
    ↓
  After 30 days: Hard DELETE leads (cron)
    ↓
  Analyses hard-deleted (CASCADE foreign key)
    ↓
  credit_ledger.lead_id → NULL (SET NULL preserves audit)
    ↓
  Transaction record intact: "2 credits charged for [deleted lead]"
```

---

## **11. DEDUPLICATION: Application-Level Check**

### **No Database Constraint** - Flexible Business Logic

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

// Proceed with analysis creation
```

### **Why No DB Constraint:**
- ✅ Allows re-analysis after completion (track growth over time)
- ✅ Supports future features: scheduled re-analysis, comparison mode
- ✅ Easier to change business rules without migrations
- ✅ Application has more context (can show friendly error message)

---

## **12. PLANS AS DATA (Not Hardcoded)**

### **Benefits:**
- ✅ Change credits without code deploy ("Black Friday: 50 free credits instead of 25!")
- ✅ Add new plans without schema changes (just INSERT)
- ✅ Feature flags per plan in JSONB
- ✅ A/B test pricing (create test plans, assign to specific users)

### **Example: Changing Credits:**
```sql
-- Marketing decides to give Pro users 150 credits instead of 100
UPDATE plans SET credits_per_month = 150 WHERE id = 'pro';

-- Next month's cron renewal automatically grants 150 credits (uses plans table)
```

### **Example: Adding New Plan:**
```sql
-- Launch "Starter" tier between Free and Pro
INSERT INTO plans VALUES (
  'starter',           -- id
  'Starter Plan',      -- name
  50,                  -- credits_per_month
  1500,                -- price_cents ($15)
  NULL,                -- stripe_price_id (fill after creating in Stripe)
  '{"max_businesses": 2}'  -- features
);
```

### **Example: Changing Price:**
```sql
-- Pro goes from $30 to $35
UPDATE plans SET price_cents = 3500 WHERE id = 'pro';

-- BUT: This doesn't affect existing Stripe subscriptions!
-- You need to:
-- 1. Create NEW Stripe price: $35/month
-- 2. Update plans table with new stripe_price_id
-- 3. Old subscriptions continue at $30 (grandfathered)
-- 4. New subscriptions use $35
```

---

## **13. NAMING CONVENTIONS (Strict Standards)**

| Pattern | Examples | Rationale |
|---------|----------|-----------|
| Timestamps end in `_at` | `created_at`, `deleted_at`, `completed_at` | Consistency, clearly a timestamp |
| `status` = mutable state | `'pending'`, `'active'`, `'canceled'` | Lifecycle changes expected |
| `{thing}_type` = category | `plan_type`, `analysis_type`, `transaction_type` | Immutable classification |
| Money in cents | `price_cents`, `cost_cents` | No decimals, no rounding errors |
| Duration in milliseconds | `processing_duration_ms` | Explicit unit in column name |
| Booleans use `is_` | `is_verified`, `is_active`, `is_suspended` | Clear boolean identification |
| Primary key always `id` | accounts(id), leads(id) | NOT `account_id` or `lead_id` in same table |

---

## **14. PERFORMANCE INDEXES (Core Set)**

### **Critical for Webhook Deduplication:**
```sql
CREATE UNIQUE INDEX idx_webhook_stripe_id ON webhook_events(stripe_event_id);
```

### **Dashboard Queries:**
```sql
CREATE INDEX idx_leads_active ON leads(account_id, last_analyzed_at DESC) 
WHERE deleted_at IS NULL;

CREATE INDEX idx_analyses_latest ON analyses(lead_id, completed_at DESC) 
WHERE deleted_at IS NULL;
```

### **Deduplication Check:**
```sql
CREATE INDEX idx_analyses_in_progress ON analyses(lead_id, status) 
WHERE status IN ('pending', 'processing') AND deleted_at IS NULL;
```

### **Credit Balance (Materialized):**
```sql
CREATE INDEX idx_credit_ledger_account ON credit_ledger(account_id, created_at DESC);
CREATE INDEX idx_credit_balances_account ON credit_balances(account_id);
```

### **Active Subscription:**
```sql
CREATE INDEX idx_subscriptions_active ON subscriptions(account_id, status) 
WHERE status = 'active';

CREATE UNIQUE INDEX idx_one_active_subscription ON subscriptions(account_id) 
WHERE status = 'active';
```

### **Account Membership (RLS Helper):**
```sql
CREATE INDEX idx_account_members_user ON account_members(user_id, account_id);
```

### **AI Usage Logs:**
```sql
CREATE INDEX idx_ai_usage_account ON ai_usage_logs(account_id, created_at DESC);
CREATE INDEX idx_ai_usage_provider ON ai_usage_logs(provider, created_at DESC);
```

---

## **15. ROW-LEVEL SECURITY (RLS) STRATEGY**

### **Security Model:**
- ✅ All user-facing tables have RLS enabled
- ✅ Users only access their own account data
- ✅ Soft-deleted records hidden by default
- ✅ Service role bypasses all policies (Cloudflare Worker)
- ✅ Admin users bypass all policies (support/debugging)
- ✅ No anonymous access (authentication required)

### **Helper Functions:**

#### **Function: auth.user_account_ids()**
```sql
-- Returns all account IDs the current user belongs to
-- Excludes deleted and suspended accounts
CREATE FUNCTION auth.user_account_ids()
RETURNS SETOF uuid
AS $$
  SELECT am.account_id 
  FROM account_members am
  JOIN accounts a ON a.id = am.account_id
  WHERE am.user_id = auth.uid()
    AND a.deleted_at IS NULL
    AND a.is_suspended = false
$$;
```

#### **Function: auth.is_admin()**
```sql
-- Returns true if current user is admin (and not suspended)
CREATE FUNCTION auth.is_admin()
RETURNS boolean
AS $$
  SELECT EXISTS (
    SELECT 1 FROM users 
    WHERE id = auth.uid() 
    AND is_admin = true
    AND is_suspended = false
  )
$$;
```

### **Policy Pattern (Applied to All User-Facing Tables):**

```sql
-- Users see data for accounts they belong to
CREATE POLICY "account_access" ON leads
FOR ALL
USING (
  auth.is_admin()  -- Admins see everything
  OR (
    account_id IN (SELECT auth.user_account_ids())
    AND deleted_at IS NULL
  )
);

-- Service role (Cloudflare Worker) bypasses RLS
CREATE POLICY "service_role_bypass" ON leads
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

### **Tables with RLS:**
- accounts (strict - only user's accounts)
- account_members (owners manage members)
- leads (account-scoped, active only)
- analyses (account-scoped, active only)
- business_profiles (account-scoped, active only)
- credit_ledger (read-only for users)
- credit_balances (read-only for users)
- account_usage_summary (read-only for users)
- subscriptions (read-only for users)
- stripe_invoices (read-only for users)

### **Tables WITHOUT RLS (Service Role Only):**
- webhook_events (Stripe processing only)
- ai_usage_logs (aggregated in platform_metrics_daily)
- platform_metrics_daily (admin dashboard only)
- plans (public read access for pricing page)

---

## **16. DATA RETENTION & ARCHIVAL STRATEGY**

### **Hot Data (Fast queries, recent):**
```
ai_usage_logs: 90 days rolling window
├── Queried for: Real-time performance metrics
├── Partitioned by month (PostgreSQL native)
└── ~13.5M rows max at scale (assuming 5000 analyses/day)
```

### **Warm Data (Aggregated, all history):**
```
account_usage_summary: Forever (monthly rollups)
platform_metrics_daily: Forever (daily rollups)
credit_ledger: Forever (audit requirement, legal compliance)
subscriptions: Forever (append-only history)
```

### **Cold Data (Archived, rarely accessed):**
```
ai_usage_logs older than 90 days:
├── Export to S3/Glacier (before deletion)
├── Parquet format (compressed, queryable)
└── Purge from database (daily cron)
```

### **Cleanup Cron Jobs:**

#### **Daily (2 AM UTC):**
```sql
-- Delete AI logs older than 90 days
DELETE FROM ai_usage_logs WHERE created_at < NOW() - INTERVAL '90 days';

-- Hard-delete soft-deleted records older than 30 days
DELETE FROM leads WHERE deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM analyses WHERE deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM business_profiles WHERE deleted_at < NOW() - INTERVAL '30 days';
DELETE FROM accounts WHERE deleted_at < NOW() - INTERVAL '30 days';
```

#### **Monthly (1st of month, 3 AM UTC):**
```sql
-- Freeze last month's usage summaries
UPDATE account_usage_summary 
SET is_finalized = true
WHERE period_end = DATE_TRUNC('month', NOW() - INTERVAL '1 day');

-- Grant monthly credits to all active subscriptions
-- (via get_renewable_subscriptions() function + deduct_credits())
```

---

# 📋 PART 4: BUSINESS LOGIC FUNCTIONS

## **Function: deduct_credits()**
**Purpose:** Atomically deduct/grant credits (prevents race conditions)

### **Signature:**
```sql
deduct_credits(
  p_account_id uuid,
  p_amount integer,          -- Negative for deductions, positive for grants
  p_transaction_type text,
  p_description text,
  p_metadata jsonb DEFAULT '{}'::jsonb
) RETURNS uuid  -- Returns transaction ID
```

### **What It Does:**
1. **Locks** credit_balances row (prevents concurrent modifications)
2. **Checks** current balance
3. **Validates** no overdraft (balance + amount >= 0)
4. **Inserts** into credit_ledger with calculated balance_after
5. **Updates** credit_balances.current_balance (materialized view)
6. **Returns** transaction ID (for reference in analyses table)

### **Usage Examples:**
```javascript
// Deduct 2 credits for deep analysis
const txId = await db.rpc('deduct_credits', {
  p_account_id: accountId,
  p_amount: -2,  // Negative = deduction
  p_transaction_type: 'analysis',
  p_description: `Deep analysis of @${username}`,
  p_metadata: { 
    analysis_id: analysisId, 
    lead_id: leadId,
    analysis_type: 'deep'
  }
});
// Throws exception if insufficient credits

// Grant 100 credits for monthly renewal
await db.rpc('deduct_credits', {
  p_account_id: accountId,
  p_amount: 100,  // Positive = grant
  p_transaction_type: 'subscription_renewal',
  p_description: 'Monthly Pro plan renewal'
});

// Admin manually adds 50 credits
await db.rpc('deduct_credits', {
  p_account_id: accountId,
  p_amount: 50,
  p_transaction_type: 'admin_grant',
  p_description: 'Compensation for downtime'
});
```

### **Why Function (Not Direct INSERT):**
- ✅ Prevents race conditions (two analyses starting simultaneously)
- ✅ Atomic balance calculation (no SUM() queries)
- ✅ Overdraft protection (transaction fails if insufficient credits)
- ✅ Consistent error messages
- ✅ Audit trail (all movements logged)

---

## **Function: generate_slug()** (continued)
**Purpose:** Create URL-safe slugs with collision handling

### **Signature:**
```sql
generate_slug(input_text text) RETURNS text
```

### **What It Does:**
1. **Lowercase** input text
2. **Replace** non-alphanumeric with hyphens
3. **Remove** leading/trailing hyphens
4. **Truncate** to 50 characters
5. **Ensure uniqueness** (append -2, -3, etc. if collision)

### **Usage Examples:**
```javascript
const slug = await db.rpc('generate_slug', { 
  input_text: 'Hamza Williams'
});
// Returns: "hamza-williams"

// If "hamza-williams" exists:
// Returns: "hamza-williams-2"

// Special characters:
generate_slug('John\'s Agency & Co!')
// Returns: "john-s-agency-co"

// Long names (truncated):
generate_slug('The Very Long Business Name That Exceeds Fifty Characters')
// Returns: "the-very-long-business-name-that-exceeds-fif"
```

### **Why Function (Not Application Logic):**
- ✅ Atomic uniqueness check (prevents race conditions)
- ✅ Consistent slug generation (same input = same output)
- ✅ Database handles collision detection
- ✅ Reusable (accounts, business profiles, future entities)

---

## **Function: get_renewable_subscriptions()**
**Purpose:** Find subscriptions that need monthly credit grants

### **Signature:**
```sql
get_renewable_subscriptions() RETURNS TABLE (
  id uuid,
  account_id uuid,
  plan_type text,
  credits_per_month integer,
  current_period_end timestamptz
)
```

### **What It Does:**
Returns all active subscriptions where:
- Status = 'active'
- current_period_end < NOW() (period has ended)
- Account is not deleted or suspended

### **Usage (Monthly Cron):**
```javascript
// Run on 1st of month at 3 AM UTC
const subscriptions = await db.rpc('get_renewable_subscriptions');

for (const sub of subscriptions) {
  // Grant credits
  await db.rpc('deduct_credits', {
    p_account_id: sub.account_id,
    p_amount: sub.credits_per_month,  // Positive = grant
    p_transaction_type: 'subscription_renewal',
    p_description: `Monthly ${sub.plan_type} renewal`
  });
  
  // Update subscription period
  const newPeriodEnd = new Date(sub.current_period_end);
  newPeriodEnd.setMonth(newPeriodEnd.getMonth() + 1);
  
  await db.query(`
    UPDATE subscriptions
    SET current_period_start = $1,
        current_period_end = $2
    WHERE id = $3
  `, [sub.current_period_end, newPeriodEnd, sub.id]);
}
```

### **Why Function:**
- ✅ Complex JOIN logic (subscriptions + plans + accounts)
- ✅ Reusable (manual renewal trigger, debugging)
- ✅ Testable (can run in SQL console)

---

## **Trigger: update_updated_at_column()**
**Purpose:** Auto-update updated_at timestamp on row changes

### **Applied To:**
- users
- accounts
- business_profiles
- account_usage_summary
- platform_metrics_daily
- credit_balances

### **How It Works:**
```sql
-- Generic trigger function
CREATE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Attach to tables
CREATE TRIGGER update_users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();
```

### **Usage Example:**
```sql
-- Manual update
UPDATE accounts SET name = 'New Name' WHERE id = 'acc_123';

-- updated_at automatically set to NOW()
-- No application code needed
```

---

## **Trigger: credit_ledger_update_balance()**
**Purpose:** Auto-update credit_balances on every credit_ledger INSERT

### **How It Works:**
```
1. Application calls: deduct_credits(account_id, -2, ...)
2. Function INSERTs into credit_ledger (amount=-2, balance_after=98)
3. Trigger fires AFTER INSERT
4. Trigger UPSERTs into credit_balances (current_balance=98)
5. Dashboard queries credit_balances (instant!)
```

### **Why Trigger (Not Function):**
- ✅ Automatic (no application code can forget)
- ✅ Consistent (always in sync with ledger)
- ✅ Efficient (single UPSERT per transaction)

---

# 📋 PART 5: CONSTRAINTS & VALIDATION

## **Foreign Key Constraints**

### **CASCADE Rules:**
```sql
-- Child is meaningless without parent (CASCADE)
analyses.lead_id → leads(id) ON DELETE CASCADE
account_members.account_id → accounts(id) ON DELETE CASCADE
account_members.user_id → users(id) ON DELETE CASCADE
credit_balances.account_id → accounts(id) ON DELETE CASCADE
```

**Why CASCADE:**
- If lead deleted, analyses become orphaned (meaningless)
- If account deleted, memberships become orphaned
- Clean cascading deletion

### **SET NULL Rules:**
```sql
-- Preserve audit trail (SET NULL)
credit_ledger.analysis_id → analyses(id) ON DELETE SET NULL
credit_ledger.lead_id → leads(id) ON DELETE SET NULL
analyses.business_profile_id → business_profiles(id) ON DELETE SET NULL
analyses.requested_by → users(id) ON DELETE SET NULL
```

**Why SET NULL:**
- Credit transaction must persist (audit requirement)
- After lead deleted: "2 credits charged for [deleted lead]"
- Analysis keeps result even if business profile deleted

### **RESTRICT Rules:**
```sql
-- Prevent accidental data loss (RESTRICT)
accounts.id ← subscriptions, credit_ledger, leads, etc.
users(id) ← accounts.owner_id
```

**Why RESTRICT:**
- Force soft delete first (UPDATE deleted_at)
- Admin must explicitly clean up child records
- Prevents accidental mass deletion

---

## **Unique Constraints**

### **Single-Column Unique:**
```sql
users.email UNIQUE                    -- One account per email
accounts.slug UNIQUE                  -- Unique URLs
webhook_events.stripe_event_id UNIQUE -- Idempotency
stripe_invoices.stripe_invoice_id UNIQUE
plans.id PRIMARY KEY                  -- Text primary key
```

### **Composite Unique:**
```sql
-- One membership per user per account
UNIQUE (account_id, user_id) ON account_members

-- One summary per account per month
UNIQUE (account_id, period_start) ON account_usage_summary

-- Same Instagram username allowed for different businesses
UNIQUE (account_id, business_profile_id, instagram_username) ON leads
```

### **Conditional Unique (Partial Index):**
```sql
-- Only ONE active subscription per account
CREATE UNIQUE INDEX idx_one_active_subscription
ON subscriptions(account_id)
WHERE status = 'active';

-- Explanation: User can have multiple subscriptions (history)
-- But only ONE can be status='active' at a time
```

---

## **Check Constraints**

### **Credit & Money Validation:**
```sql
-- Balances cannot be negative
CHECK (credit_balances.current_balance >= 0)
CHECK (credit_ledger.balance_after >= 0)

-- Costs cannot be negative
CHECK (ai_usage_logs.cost_cents >= 0)
CHECK (plans.price_cents >= 0)
CHECK (plans.credits_per_month >= 0)
CHECK (subscriptions.price_cents >= 0)
```

### **Enum-Style Validation:**
```sql
-- Valid subscription statuses
CHECK (subscriptions.status IN (
  'active', 
  'canceled', 
  'past_due', 
  'paused', 
  'pending_stripe_confirmation'
))

-- Valid analysis statuses
CHECK (analyses.status IN (
  'pending', 
  'processing', 
  'completed', 
  'failed'
))

-- Valid account member roles
CHECK (account_members.role IN (
  'owner', 
  'admin', 
  'member', 
  'viewer'
))

-- Valid transaction types
CHECK (credit_ledger.transaction_type IN (
  'subscription_renewal',
  'analysis',
  'refund',
  'admin_grant',
  'chargeback',
  'signup_bonus'
))

-- Valid analysis types
CHECK (analyses.analysis_type IN ('light', 'deep'))

-- Valid AI providers
CHECK (ai_usage_logs.provider IN ('openai', 'anthropic', 'apify'))

-- Valid AI call statuses
CHECK (ai_usage_logs.status IN ('success', 'error', 'timeout'))

-- Valid invoice statuses
CHECK (stripe_invoices.status IN ('paid', 'open', 'void', 'uncollectible'))
```

### **Why Check Constraints:**
- ✅ Database enforces validity (no invalid states possible)
- ✅ Application bugs can't corrupt data
- ✅ Clear error messages (constraint violation)
- ✅ Documentation (constraints = valid values)

---

## **Not Null Constraints**

### **Critical Fields That Must Exist:**
```sql
-- Identity
users.email NOT NULL
users.id NOT NULL (implicit PRIMARY KEY)

-- Accounts
accounts.owner_id NOT NULL
accounts.name NOT NULL

-- Billing
subscriptions.account_id NOT NULL
subscriptions.plan_type NOT NULL
subscriptions.current_period_start NOT NULL
subscriptions.current_period_end NOT NULL
credit_ledger.account_id NOT NULL
credit_ledger.amount NOT NULL
credit_ledger.balance_after NOT NULL

-- Business Logic
leads.account_id NOT NULL
leads.instagram_username NOT NULL
analyses.lead_id NOT NULL
analyses.account_id NOT NULL
analyses.analysis_type NOT NULL

-- Audit
ai_usage_logs.provider NOT NULL
ai_usage_logs.cost_cents NOT NULL
```

### **Nullable Fields (By Design):**
```sql
-- Optional user info
users.full_name NULL
users.signature_name NULL
users.avatar_url NULL

-- Stripe IDs (filled after webhook)
subscriptions.stripe_customer_id NULL  -- Until Stripe confirms
subscriptions.stripe_subscription_id NULL  -- NULL for free plan

-- Optional lead metadata
leads.display_name NULL  -- Profile might not have one
leads.bio NULL
leads.external_url NULL

-- Analysis results (populated after completion)
analyses.ai_response NULL  -- Until analysis completes
analyses.overall_score NULL
analyses.completed_at NULL  -- NULL while pending
```

---

# 📋 PART 6: SIGNUP & ONBOARDING FLOW

## **Complete Signup Flow (Step-by-Step)**

### **Frontend (Supabase Auth):**
```javascript
// Step 1: User fills signup form
const { data, error } = await supabase.auth.signUp({
  email: 'user@example.com',
  password: 'SecurePass123!',
  options: {
    data: { 
      full_name: 'Hamza Williams'  // Stored in auth.users metadata
    }
  }
});

if (error) {
  // Show error: "Email already registered", "Weak password", etc.
  return;
}

// Step 2: Call backend to complete setup
const response = await fetch('/api/auth/complete-signup', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: data.user.id,
    email: data.user.email,
    fullName: data.user.user_metadata.full_name
  })
});

const result = await response.json();

// Step 3: Redirect to onboarding
if (result.success) {
  window.location.href = `/accounts/${result.account.slug}/onboarding`;
}
```

### **Backend (Cloudflare Worker - POST /api/auth/complete-signup):**
```javascript
export async function completeSignup(c) {
  const { userId, email, fullName } = await c.req.json();
  
  try {
    // 1. Create Stripe customer (optimistic)
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
    // "Hamza Williams" → "hamza-williams"
    
    // 3. Create account
    const account = await db.query(`
      INSERT INTO accounts (owner_id, name, slug)
      VALUES ($1, $2, $3)
      RETURNING id
    `, [userId, `${fullName}'s Account`, slug]);
    
    const accountId = account.rows[0].id;
    
    // 4. Add user to account_members (owner role)
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
        stripe_subscription_id,  -- NULL for free
        status,
        current_period_start,
        current_period_end
      ) VALUES ($1, 'free', 0, $2, NULL, 'active', $3, $4)
    `, [accountId, customer.id, periodStart, periodEnd]);
    
    // 6. Grant 25 welcome credits (optimistic)
    await db.rpc('deduct_credits', {
      p_account_id: accountId,
      p_amount: 25,  // Positive = grant
      p_transaction_type: 'signup_bonus',
      p_description: 'Welcome to Oslira!'
    });
    
    // 7. Initialize usage summary for current month
    const now = new Date();
    await db.query(`
      INSERT INTO account_usage_summary (
        account_id,
        period_start,
        period_end
      ) VALUES ($1, $2, $3)
    `, [
      accountId,
      new Date(now.getFullYear(), now.getMonth(), 1),  // 1st of month
      new Date(now.getFullYear(), now.getMonth() + 1, 0)  // Last day of month
    ]);
    
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
    return c.json({ 
      success: false, 
      error: error.message 
    }, 500);
  }
}
```

### **What Happens After Signup:**
```
✅ User can immediately use the app (25 credits available)
✅ URL: app.oslira.com/accounts/hamza-williams/dashboard
✅ Stripe customer created (for future upgrades)
✅ Free subscription active
✅ Monthly cron will renew credits on 1st of next month

Background (async):
  - Stripe webhook confirms customer.created
  - No action needed (already have stripe_customer_id)
```

---

## **Business Profile Onboarding (After Signup)**

### **URL:** `/accounts/{slug}/onboarding`

### **Flow:**
```
Step 1: Welcome Screen
  → "Let's set up your first business profile"
  
Step 2: Basic Info
  → Business Name (e.g., "Oslira Marketing")
  → Website (e.g., "oslira.com")
  → One-liner (e.g., "We write copy that sells")
  
Step 3: Business Context (15+ fields)
  → Niche (e.g., "B2B SaaS copywriting")
  → Target Audience (e.g., "Tech founders, marketing managers")
  → Value Proposition (e.g., "Conversion-focused copy")
  → Problems Solved (e.g., "Low conversion rates, generic messaging")
  → Unique Approach (e.g., "Data-driven storytelling")
  → Ideal Client Traits (e.g., "Fast-growing startups")
  → Service Offerings (e.g., "Landing pages, email sequences")
  → Tone of Voice (e.g., "Professional yet approachable")
  → Competitors (e.g., "Agency X, Freelancer Y")
  → Geographic Focus (e.g., "North America")
  
Step 4: AI Processing
  → "Generating your business context..."
  → AI creates business_context_pack JSON
  → Stored in business_profiles.business_context_pack
  
Step 5: Complete
  → Redirect to: /accounts/{slug}/dashboard
  → Ready to analyze leads!
```

### **Backend (POST /api/business-profiles):**
```javascript
export async function createBusinessProfile(c) {
  const { accountId, formData } = await c.req.json();
  
  // 1. Validate user has access to account
  const hasAccess = await checkAccountAccess(c, accountId);
  if (!hasAccess) return c.json({ error: 'Unauthorized' }, 403);
  
  // 2. Generate AI context pack
  const contextPack = await generateBusinessContext(formData);
  // Uses OpenAI/Claude to structure the data
  
  // 3. Insert business profile
  const profile = await db.query(`
    INSERT INTO business_profiles (
      account_id,
      business_name,
      website,
      business_one_liner,
      business_context_pack,
      context_version,
      context_generated_at
    ) VALUES ($1, $2, $3, $4, $5, 'v1.0', NOW())
    RETURNING id
  `, [
    accountId,
    formData.businessName,
    formData.website,
    formData.oneLiner,
    contextPack
  ]);
  
  return c.json({
    success: true,
    profileId: profile.rows[0].id
  });
}
```

---

# 📋 PART 7: STRIPE WEBHOOK HANDLERS

## **Webhook Endpoint:** `POST /webhooks/stripe`

### **Handler Pattern:**
```javascript
export async function handleStripeWebhook(c) {
  const signature = c.req.header('stripe-signature');
  const body = await c.req.text();
  
  // 1. Verify signature (security)
  let event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      STRIPE_WEBHOOK_SECRET
    );
  } catch (error) {
    return c.json({ error: 'Invalid signature' }, 400);
  }
  
  // 2. Check idempotency (prevent duplicate processing)
  const existing = await db.query(
    'SELECT 1 FROM webhook_events WHERE stripe_event_id = $1',
    [event.id]
  );
  
  if (existing.rows.length > 0) {
    return c.json({ received: true, message: 'already_processed' });
  }
  
  // 3. Process event based on type
  switch (event.type) {
    case 'invoice.paid':
      await handleInvoicePaid(event.data.object);
      break;
      
    case 'invoice.payment_failed':
      await handleInvoiceFailed(event.data.object);
      break;
      
    case 'customer.subscription.created':
      await handleSubscriptionCreated(event.data.object);
      break;
      
    case 'customer.subscription.updated':
      await handleSubscriptionUpdated(event.data.object);
      break;
      
    case 'customer.subscription.deleted':
      await handleSubscriptionDeleted(event.data.object);
      break;
  }
  
  // 4. Record event (idempotency)
  await db.query(`
    INSERT INTO webhook_events (
      stripe_event_id,
      event_type,
      processed_at,
      payload
    ) VALUES ($1, $2, NOW(), $3)
  `, [event.id, event.type, event]);
  
  return c.json({ received: true });
}
```

---

## **Handler: invoice.paid**
```javascript
async function handleInvoicePaid(invoice) {
  // 1. Find account from Stripe customer ID
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
  
  // 2. Store invoice record
  await db.query(`
    INSERT INTO stripe_invoices (
      account_id,
      stripe_invoice_id,
      stripe_subscription_id,
      amount_cents,
      status,
      paid_at
    ) VALUES ($1, $2, $3, $4, 'paid', $5)
    ON CONFLICT (stripe_invoice_id) DO NOTHING
  `, [
    accountId,
    invoice.id,
    invoice.subscription,
    invoice.amount_paid,
    new Date(invoice.status_transitions.paid_at * 1000)
  ]);
  
  // 3. Credits already granted by Stripe subscription webhook
  // (This is just for invoice tracking)
  
  console.log('Invoice recorded:', invoice.id);
}
```

---

## **Handler: invoice.payment_failed**
```javascript
async function handleInvoiceFailed(invoice) {
  // 1. Find account
  const account = await db.query(`
    SELECT account_id FROM subscriptions 
    WHERE stripe_customer_id = $1 
    LIMIT 1
  `, [invoice.customer]);
  
  if (!account.rows.length) return;
  
  const accountId = account.rows[0].account_id;
  
  // 2. Store failed invoice
  await db.query(`
    INSERT INTO stripe_invoices (
      account_id,
      stripe_invoice_id,
      amount_cents,
      status,
      failed_at,
      failure_reason
    ) VALUES ($1, $2, $3, 'uncollectible', NOW(), $4)
    ON CONFLICT (stripe_invoice_id) DO NOTHING
  `, [
    accountId,
    invoice.id,
    invoice.amount_due,
    invoice.last_finalization_error?.message || 'Payment failed'
  ]);
  
  // 3. Update subscription status
  await db.query(`
    UPDATE subscriptions
    SET status = 'past_due'
    WHERE stripe_customer_id = $1
      AND status = 'active'
  `, [invoice.customer]);
  
  // 4. Send email notification (optional)
  // await sendPaymentFailedEmail(accountId);
  
  console.log('Payment failed for:', invoice.customer);
}
```

---

## **Handler: customer.subscription.created**
```javascript
async function handleSubscriptionCreated(subscription) {
  // This fires when user upgrades from free to paid
  
  // 1. Find pending subscription
  const existing = await db.query(`
    SELECT id FROM subscriptions
    WHERE stripe_customer_id = $1
      AND status = 'pending_stripe_confirmation'
    LIMIT 1
  `, [subscription.customer]);
  
  if (existing.rows.length > 0) {
    // 2. Update with Stripe subscription ID
    await db.query(`
      UPDATE subscriptions
      SET stripe_subscription_id = $1,
          stripe_price_id = $2,
          status = 'active'
      WHERE id = $3
    `, [
      subscription.id,
      subscription.items.data[0].price.id,
      existing.rows[0].id
    ]);
  } else {
    // 3. Create new subscription (user upgraded via Stripe dashboard)
    const account = await db.query(`
      SELECT account_id FROM subscriptions
      WHERE stripe_customer_id = $1
      LIMIT 1
    `, [subscription.customer]);
    
    if (!account.rows.length) return;
    
    await db.query(`
      INSERT INTO subscriptions (
        account_id,
        plan_type,
        price_cents,
        stripe_customer_id,
        stripe_subscription_id,
        stripe_price_id,
        status,
        current_period_start,
        current_period_end
      ) VALUES ($1, $2, $3, $4, $5, $6, 'active', $7, $8)
    `, [
      account.rows[0].account_id,
      getPlanTypeFromPrice(subscription.items.data[0].price.id),
      subscription.items.data[0].price.unit_amount,
      subscription.customer,
      subscription.id,
      subscription.items.data[0].price.id,
      new Date(subscription.current_period_start * 1000),
      new Date(subscription.current_period_end * 1000)
    ]);
  }
  
  console.log('Subscription created:', subscription.id);
}
```

---

## **Handler: customer.subscription.updated**
```javascript
async function handleSubscriptionUpdated(subscription) {
  // Update subscription status (active, past_due, canceled, etc.)
  
  await db.query(`
    UPDATE subscriptions
    SET status = $1,
        current_period_start = $2,
        current_period_end = $3,
        canceled_at = $4
    WHERE stripe_subscription_id = $5
  `, [
    subscription.status,
    new Date(subscription.current_period_start * 1000),
    new Date(subscription.current_period_end * 1000),
    subscription.canceled_at ? new Date(subscription.canceled_at * 1000) : null,
    subscription.id
  ]);
  
  console.log('Subscription updated:', subscription.id, subscription.status);
}
```

---

## **Handler: customer.subscription.deleted**
```javascript
async function handleSubscriptionDeleted(subscription) {
  // Subscription ended (canceled or payment failed repeatedly)
  
  await db.query(`
    UPDATE subscriptions
    SET status = 'canceled',
        canceled_at = NOW()
    WHERE stripe_subscription_id = $1
  `, [subscription.id]);
  
  // User keeps existing credits (they don't expire)
  // Just won't get monthly renewals anymore
  
  console.log('Subscription deleted:', subscription.id);
}
```

---

# 📋 PART 8: CRON JOBS

## **Daily Cleanup Job** (2 AM UTC)

### **Purpose:**
- Delete AI usage logs older than 90 days
- Hard-delete soft-deleted records older than 30 days
- Prevent database bloat

### **Cloudflare Worker Cron:**
```javascript
// wrangler.toml
[triggers]
crons = ["0 2 * * *"]  // Daily at 2 AM UTC

// src/cron/daily-cleanup.js
export async function dailyCleanup(env) {
  const db = getDatabase(env);
  
  try {
    // 1. Delete old AI logs (90 days)
    const aiLogsResult = await db.query(`
      DELETE FROM ai_usage_logs 
      WHERE created_at < NOW() - INTERVAL '90 days'
    `);
    console.log(`Deleted ${aiLogsResult.rowCount} AI logs`);
    
    // 2. Hard-delete old leads (30 days soft-deleted)
    const leadsResult = await db.query(`
      DELETE FROM leads 
      WHERE deleted_at < NOW() - INTERVAL '30 days'
    `);
    console.log(`Hard-deleted ${leadsResult.rowCount} leads`);
    // Note: Analyses CASCADE deleted automatically
    
    // 3. Hard-delete old business profiles (30 days)
    const profilesResult = await db.query(`
      DELETE FROM business_profiles 
      WHERE deleted_at < NOW() - INTERVAL '30 days'
    `);
    console.log(`Hard-deleted ${profilesResult.rowCount} profiles`);
    
    // 4. Hard-delete old accounts (30 days)
    const accountsResult = await db.query(`
      DELETE FROM accounts 
      WHERE deleted_at < NOW() - INTERVAL '30 days'
    `);
    console.log(`Hard-deleted ${accountsResult.rowCount} accounts`);
    
    return {
      success: true,
      deleted: {
        aiLogs: aiLogsResult.rowCount,
        leads: leadsResult.rowCount,
        profiles: profilesResult.rowCount,
        accounts: accountsResult.rowCount
      }
    };
    
  } catch (error) {
    console.error('Daily cleanup failed:', error);
    throw error;
  }
}
```

---

## **Monthly Rollup Job** (1st of month, 3 AM UTC)

### **Purpose:**
- Finalize previous month's usage summaries
- Grant monthly credits to all active subscriptions
- Generate monthly platform metrics

### **Cloudflare Worker Cron:**
```javascript
// wrangler.toml
[triggers]
crons = ["0 3 1 * *"]  // 1st of month at 3 AM UTC

// src/cron/monthly-rollup.js
export async function monthlyRollup(env) {
  const db = getDatabase(env);
  
  try {
    // 1. Finalize last month's usage summaries
    const lastMonthEnd = new Date();
    lastMonthEnd.setDate(0); // Last day of previous month
    
    await db.query(`
      UPDATE account_usage_summary
      SET is_finalized = true
      WHERE period_end = $1
        AND is_finalized = false
    `, [lastMonthEnd.toISOString().split('T')[0]]);
    
    console.log('Finalized last month summaries');
    
    // 2. Get all renewable subscriptions
    const subscriptions = await db.rpc('get_renewable_subscriptions');
    
    console.log(`Found ${subscriptions.length} subscriptions to renew`);
    
    // 3. Grant credits to each subscription
    let renewedCount = 0;
    
    for (const sub of subscriptions) {
      try {
        // Grant credits
        await db.rpc('deduct_credits', {
          p_account_id: sub.account_id,
          p_amount: sub.credits_per_month,  // Positive = grant
          p_transaction_type: 'subscription_renewal',
          p_description: `Monthly ${sub.plan_type} renewal`,
          p_metadata: {
            subscription_id: sub.id,
            plan_type: sub.plan_type
          }
        });
        
        // Update subscription period
        const newPeriodEnd = new Date(sub.current_period_end);
        newPeriodEnd.setMonth(newPeriodEnd.getMonth() + 1);
        
        await db.query(`
          UPDATE subscriptions
          SET current_period_start = $1,
              current_period_end = $2
          WHERE id = $3
        `, [sub.current_period_end, newPeriodEnd, sub.id]);
        
        renewedCount++;
        
      } catch (error) {
        console.error(`Failed to renew ${sub.id}:`, error);
        // Continue with other subscriptions
      }
    }
    
    console.log(`Renewed ${renewedCount} subscriptions`);
    
    return {
      success: true,
      renewed: renewedCount
    };
    
  } catch (error) {
    console.error('Monthly rollup failed:', error);
    throw error;
  }
}
```

---

# 📋 PART 9: IMPLEMENTATION CHECKLIST

## **Phase 1: Database Setup** (Day 1)

### **Step 1: Create Supabase Project**
- [ ] Go to https://supabase.com/dashboard
- [ ] Click "New Project"
- [ ] Name: "oslira-production"
- [ ] Choose region (closest to users)
- [ ] Save project URL and keys:
  - Project URL: `https://xxx.supabase.co`
  - `anon` key (for client)
  - `service_role` key (for Worker)

### **Step 2: Execute Schema SQL** (3 separate files)
- [ ] File 1: **01-create-tables.sql** (all CREATE TABLE statements)
- [ ] File 2: **02-add-constraints-indexes-functions-triggers.sql** (everything else except RLS)
- [ ] File 3: **03-enable-rls.sql** (all RLS policies)

Run in Supabase SQL Editor in this exact order.

### **Step 3: Preseed Data**
- [ ] File 4: **04-preseed.sql** (INSERT plans data)

### **Step 4: Verify Deployment**
```sql
-- Check all 15 tables created
SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'public'
ORDER BY table_name;

-- Verify plans inserted
SELECT * FROM plans;

-- Verify functions exist
SELECT routine_name 
FROM information_schema.routines 
WHERE routine_schema = 'public';
```

---

## **Phase 2: AWS Secrets Manager** (Day 1)

### **Step 1: Create IAM User**
- [ ] AWS Console → IAM → Users → Create User
- [ ] Name: `oslira-worker`
- [ ] Permissions: `SecretsManagerReadWrite`
- [ ] Save: Access Key ID + Secret Access Key

### **Step 2: Store Secrets**
```bash
aws secretsmanager create-secret \
  --name oslira/production \
  --secret-string '{
    "SUPABASE_URL": "https://xxx.supabase.co",
    "SUPABASE_SERVICE_ROLE": "eyJhbGc...",
    "STRIPE_SECRET_KEY": "sk_live_...",
    "STRIPE_WEBHOOK_SECRET": "whsec_...",
    "OPENAI_API_KEY": "sk-...",
    "ANTHROPIC_API_KEY": "sk-ant-...",
    "APIFY_API_KEY": "apify_api_..."
  }'
```

---

## **Phase 3: Stripe Setup** (Day 1-2)

### **Step 1: Create Products**
- [ ] Go to https://dashboard.stripe.com/products
- [ ] Create "Pro Plan": $30/month → Copy price ID
- [ ] Create "Agency Plan": $80/month → Copy price ID
- [ ] Create "Enterprise Plan": $300/month → Copy price ID

### **Step 2: Update Plans Table**
```sql
UPDATE plans SET stripe_price_id = 'price_xxx' WHERE id = 'pro';
UPDATE plans SET stripe_price_id = 'price_yyy' WHERE id = 'agency';
UPDATE plans SET stripe_price_id = 'price_zzz' WHERE id = 'enterprise';
```

### **Step 3: Create Webhook**
- [ ] Go to https://dashboard.stripe.com/webhooks
- [ ] Add endpoint: `https://api.oslira.com/webhooks/stripe`
- [ ] Select events:
  - `invoice.paid`
  - `invoice.payment_failed`
  - `customer.subscription.created`
  - `customer.subscription.updated`
  - `customer.subscription.deleted`
- [ ] Copy webhook signing secret
- [ ] Update AWS secret with webhook secret

---

## **Phase 4: Cloudflare Worker Deploy** (Day 2)

### **Step 1: Configure wrangler.toml**
```toml
name = "oslira-worker"
main = "src/index.ts"

[vars]
APP_ENV = "production"
AWS_REGION = "us-east-1"

[triggers]
crons = [
  "0 2 * * *",    # Daily cleanup
  "0 3 1 * *"     # Monthly rollup
]
```

### **Step 2: Add AWS Credentials**
```bash
npx wrangler secret put AWS_ACCESS_KEY_ID
# Paste: (from IAM user)

npx wrangler secret put AWS_SECRET_ACCESS_KEY
# Paste: (from IAM user)
```

### **Step 3: Deploy**
```bash
npm install
npm run build
npx wrangler deploy
```

---

## **Phase 5: First Admin User** (Day 2)

### **Step 1: Sign Up**
- [ ] Go to app.oslira.com/signup
- [ ] Complete signup with your email
- [ ] Verify email

### **Step 2: Grant Admin**
```sql
-- In Supabase SQL Editor
SELECT id, email FROM auth.users WHERE email = 'your-email@example.com';

UPDATE users 
SET is_admin = true 
WHERE id = 'paste-uuid-here';
```

---

## **Phase 6: Smoke Tests** (Day 2-3)

### **Test 1: Signup Flow**
- [ ] Sign up new test user
- [ ] Verify account created
- [ ] Verify 25 credits granted
- [ ] Verify free subscription active

### **Test 2: Credit Deduction**
- [ ] Trigger analysis as test user
- [ ] Verify credits deducted (25 → 23)
- [ ] Verify credit_ledger row created
- [ ] Verify credit_balances updated

### **Test 3: RLS Policies**
```sql
-- Login as test user (non-admin)
SET request.jwt.claims.sub = 'test-user-uuid';

-- Try to see all accounts
SELECT * FROM accounts;
-- Should return ONLY test user's account

-- Try to see admin's leads
SELECT * FROM leads WHERE account_id = 'admin-account-uuid';
-- Should return EMPTY
```

### **Test 4: Webhook Processing**
- [ ] Trigger test webhook from Stripe dashboard
- [ ] Check Cloudflare Worker logs
- [ ] Verify webhook_events row created
- [ ] Verify no duplicate processing on retry

### **Test 5: Cron Jobs**
```bash
# Manually trigger daily cleanup
curl -X POST https://api.oslira.com/__scheduled \
  -H "Content-Type: application/json" \
  -d '{"cron": "0 2 * * *"}'

# Check logs for success
```

---

# 📋 PART 10: QUERIES FOR APPLICATION CODE

## **Common Queries Your Application Will Need**

### **Get User's Accounts**
```sql
-- Returns all accounts user belongs to
SELECT 
  a.id,
  a.name,
  a.slug,
  am.role,
  cb.current_balance as credits
FROM accounts a
JOIN account_members am ON am.account_id = a.id
LEFT JOIN credit_balances cb ON cb.account_id = a.id
WHERE am.user_id = $1
  AND a.deleted_at IS NULL
  AND a.is_suspended = false
ORDER BY a.created_at DESC;
```

### **Get Account Details with Subscription**
```sql
SELECT 
  a.*,
  s.plan_type,
  s.status as subscription_status,
  s.current_period_end,
  cb.current_balance as credits,
  p.credits_per_month
FROM accounts a
LEFT JOIN subscriptions s ON s.account_id = a.id AND s.status = 'active'
LEFT JOIN plans p ON p.id = s.plan_type
LEFT JOIN credit_balances cb ON cb.account_id = a.id
WHERE a.id = $1
  AND a.deleted_at IS NULL;
```

### **Get Business Profiles for Account**
```sql
SELECT *
FROM business_profiles
WHERE account_id = $1
  AND deleted_at IS NULL
ORDER BY created_at DESC;
```

### **Get Leads for Business Profile (with Latest Analysis)**
```sql
SELECT 
  l.*,
  a.id as latest_analysis_id,
  a.overall_score,
  a.niche_fit_score,
  a.completed_at as last_analyzed
FROM leads l
LEFT JOIN LATERAL (
  SELECT id, overall_score, niche_fit_score, completed_at
  FROM analyses
  WHERE lead_id = l.id
    AND status = 'completed'
    AND deleted_at IS NULL
  ORDER BY completed_at DESC
  LIMIT 1
) a ON true
WHERE l.business_profile_id = $1
  AND l.deleted_at IS NULL
ORDER BY l.last_analyzed_at DESC NULLS LAST;
```

### **Check if Analysis Already In Progress**
```sql
-- Before creating new analysis
SELECT id, status
FROM analyses
WHERE lead_id = $1
  AND status IN ('pending', 'processing')
  AND deleted_at IS NULL
LIMIT 1;

-- If rows returned: show error "Analysis already in progress"
```

### **Get Credit Transaction History**
```sql
SELECT 
  cl.*,
  u.full_name as created_by_name
FROM credit_ledger cl
LEFT JOIN users u ON u.id = cl.created_by
WHERE cl.account_id = $1
ORDER BY cl.created_at DESC
LIMIT 50;
```

### **Get Monthly Usage Stats**
```sql
SELECT *
FROM account_usage_summary
WHERE account_id = $1
ORDER BY period_start DESC
LIMIT 12;  -- Last 12 months
```

### **Get Platform Metrics (Admin Only)**
```sql
SELECT *
FROM platform_metrics_daily
ORDER BY metric_date DESC
LIMIT 90;  -- Last 90 days
```

---

## **Common Mutations Your Application Will Need**

### **Create Analysis**
```javascript
// 1. Check for in-progress analysis
const existing = await db.query(`
  SELECT id FROM analyses 
  WHERE lead_id = $1 AND status IN ('pending', 'processing')
`, [leadId]);

if (existing.rows.length > 0) {
  throw new Error('Analysis already in progress');
}

// 2. Deduct credits (throws if insufficient)
const txId = await db.rpc('deduct_credits', {
  p_account_id: accountId,
  p_amount: analysisType === 'deep' ? -2 : -1,
  p_transaction_type: 'analysis',
  p_description: `${analysisType} analysis of @${username}`
});

// 3. Create analysis record
const analysis = await db.query(`
  INSERT INTO analyses (
    lead_id,
    account_id,
    business_profile_id,
    requested_by,
    analysis_type,
    status
  ) VALUES ($1, $2, $3, $4, $5, 'pending')
  RETURNING id
`, [leadId, accountId, businessProfileId, userId, analysisType]);

// 4. Queue for background processing
await queueAnalysis(analysis.rows[0].id);
```

### **Update Lead Profile Data (Re-Analysis)**
```javascript
// When re-analyzing, PATCH the lead record
await db.query(`
  UPDATE leads
  SET follower_count = $1,
      bio = $2,
      profile_pic_url = $3,
      last_analyzed_at = NOW()
  WHERE id = $4
`, [followerCount, bio, profilePicUrl, leadId]);
```

### **Soft Delete Lead**
```javascript
await db.query(`
  UPDATE leads
  SET deleted_at = NOW()
  WHERE id = $1
`, [leadId]);

// Also soft-delete related analyses
await db.query(`
  UPDATE analyses
  SET deleted_at = NOW()
  WHERE lead_id = $1
`, [leadId]);
```

### **Restore Soft-Deleted Lead**
```javascript
await db.query(`
  UPDATE leads
  SET deleted_at = NULL
  WHERE id = $1
    AND deleted_at > NOW() - INTERVAL '30 days'
`, [leadId]);

// Also restore analyses
await db.query(`
  UPDATE analyses
  SET deleted_at = NULL
  WHERE lead_id = $1
`, [leadId]);
```

---

# 📋 PART 11: DATA TYPES & IMPORTANT NOTES

## **Critical Data Type Warnings**

### **Stripe IDs are TEXT (not UUID):**
```sql
-- ❌ WRONG
stripe_customer_id uuid
stripe_subscription_id uuid

-- ✅ CORRECT
stripe_customer_id text  -- Stripe returns "cus_abc123..."
stripe_subscription_id text  -- Stripe returns "sub_xyz789..."
stripe_price_id text  -- Stripe returns "price_1ABC..."
stripe_event_id text  -- Stripe returns "evt_1A2B3C..."
stripe_invoice_id text  -- Stripe returns "in_1DEF456..."
```

### **Money Always in Cents (Integer):**
```sql
-- ❌ WRONG
price decimal(10,2)  -- Can cause rounding errors

-- ✅ CORRECT
price_cents integer  -- $97.00 = 9700 cents
```

### **Timestamps Always timestamptz:**
```sql
-- ❌ WRONG
created_at timestamp  -- No timezone info

-- ✅ CORRECT
created_at timestamptz  -- With timezone
```

---

## **Query Pattern Best Practices**

### **Always Filter Deleted Records:**
```sql
-- ❌ WRONG
SELECT * FROM leads WHERE account_id = $1;

-- ✅ CORRECT
SELECT * FROM leads 
WHERE account_id = $1 
  AND deleted_at IS NULL;
```

### **Always Use Parameterized Queries:**
```javascript
// ❌ WRONG (SQL injection risk)
const sql = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ CORRECT
const result = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);
```

### **Use RPC for Complex Operations:**
```javascript
// ❌ WRONG (race condition)
const balance = await db.query('SELECT SUM(amount) FROM credit_ledger WHERE account_id = $1', [accountId]);
if (balance.rows[0].sum >= 2) {
  await db.query('INSERT INTO credit_ledger ...', [-2, ...]);
}

// ✅ CORRECT (atomic)
await db.rpc('deduct_credits', {
  p_account_id: accountId,
  p_amount: -2,
  ...
});
```

---

# 📋 PART 12: FUTURE ENHANCEMENTS

## **Phase 2 Features** (Post-Launch)

### **1. Team Permissions** (Role-Based Access)
- Implement role enforcement in RLS policies
- Owner: Full control
- Admin: Manage members/viewers
- Member: Create analyses, spend credits
- Viewer: Read-only

### **2. Team Member Invitations**
- Invite via email
- Pending invitations table
- Accept/decline flow

### **3. Trash/Restore UI**
- View deleted leads within 30 days
- Restore button
- Permanent delete button

### **4. Credit Purchase (Top-Up)**
- Buy additional credits without upgrading
- Stripe one-time payment
- Credit packs (50 credits for $15, etc.)

### **5. Usage Analytics Dashboard**
- Monthly credit consumption charts
- Analysis success rate
- Average lead score over time
- Cost per analysis

### **6. Webhook Retry Logic**
- Exponential backoff for failed webhooks
- Dead letter queue
- Manual retry UI for admins

### **7. Multi-Currency Support**
- Detect user location
- Show prices in local currency
- Stripe handles conversion

### **8. Annual Subscriptions**
- 2-month discount (10 months for price of 12)
- One-time payment
- Credits granted upfront

---

## **Phase 3 Features** (3-6 Months)

### **1. TikTok/YouTube Support**
- Additional platforms in leads.platform column
- Platform-specific scraping
- Cross-platform lead tracking

### **2. Scheduled Re-Analysis**
- Cron: "Re-analyze every 30 days"
- Email notification when score changes
- Track growth over time

### **3. Lead Scoring Filters**
- Filter by score range (80-100)
- Filter by niche fit
- Saved filters

### **4. Export Leads**
- CSV export
- Include analysis results
- Filtered exports

### **5. API Access (Enterprise)**
- REST API for lead data
- Webhook callbacks on analysis completion
- API key management

### **6. White-Label (Enterprise)**
- Custom domain
- Custom branding
- Hide "Powered by Oslira"

---

# 📋 PART 13: TROUBLESHOOTING GUIDE

## **Common Issues & Solutions**

### **Issue: "Insufficient credits" error**
**Cause:** User tried to analyze with < 2 credits

**Solution:**
```sql
-- Check actual balance
SELECT current_balance FROM credit_balances WHERE account_id = 'xxx';

-- Check ledger (audit)
SELECT * FROM credit_ledger WHERE account_id = 'xxx' ORDER BY created_at DESC LIMIT 10;

-- If mismatch: Rebuild credit_balances
DELETE FROM credit_balances WHERE account_id = 'xxx';
-- Trigger will recreate on next credit_ledger INSERT
```

---

### **Issue: "Analysis stuck in 'processing' status"**
**Cause:** Worker crashed mid-analysis

**Solution:**
```sql
-- Find stuck analyses
SELECT * FROM analyses 
WHERE status = 'processing' 
  AND started_at < NOW() - INTERVAL '10 minutes';

-- Mark as failed
UPDATE analyses
SET status = 'failed',
    completed_at = NOW()
WHERE id = 'xxx';

-- Refund credits
SELECT * FROM deduct_credits(
  'account_id',
  2,  -- Refund 2 credits
  'refund',
  'Analysis timeout refund'
);
```

---

### **Issue: "Duplicate webhook processing"**
**Cause:** Race condition in webhook handler

**Solution:**
```sql
-- Check for duplicates
SELECT stripe_event_id, COUNT(*) 
FROM webhook_events 
GROUP BY stripe_event_id 
HAVING COUNT(*) > 1;

-- If found: Check application logs
-- Ensure UNIQUE constraint on webhook_events.stripe_event_id
```

---

### **Issue: "RLS policy blocking service role"**
**Cause:** Missing service role bypass policy

**Solution:**
```sql
-- Add to affected table
CREATE POLICY "service_role_bypass" ON table_name
FOR ALL
USING (auth.jwt() ->> 'role' = 'service_role');
```

---

### **Issue: "User can't see their own data"**
**Cause:** Not in account_members or account suspended

**Solution:**
```sql
-- Check membership
SELECT * FROM account_members WHERE user_id = 'xxx';

-- Check account status
SELECT id, is_suspended, deleted_at 
FROM accounts 
WHERE id IN (SELECT account_id FROM account_members WHERE user_id = 'xxx');

-- If suspended: Unsuspend
UPDATE accounts SET is_suspended = false, suspended_at = NULL WHERE id = 'xxx';
```

---

**END OF MASTER PLAN**

---

## 🎯 NEXT STEPS

Once this plan is approved:

1. **Generate SQL Files:**
   - `01-create-tables.sql` (all CREATE TABLE statements)
   - `02-add-constraints-indexes-functions-triggers.sql`
   - `03-enable-rls.sql`
   - `04-preseed.sql`

2. **Generate Application Code Examples:**
   - Signup flow (complete implementation)
   - Analysis flow (complete implementation)
   - Webhook handlers (complete implementation)

3. **Generate Testing Suite:**
   - Unit tests for RLS policies
   - Integration tests for credit deduction
   - Webhook idempotency tests

**Status:** ✅ Plan Complete - Ready for Implementation
// Returns: "hamza-williams
