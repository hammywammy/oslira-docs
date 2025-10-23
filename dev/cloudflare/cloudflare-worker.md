# 🏗️ OSLIRA CLOUDFLARE WORKER - COMPLETE IMPLEMENTATION SPECIFICATION v6.0

**Status:** Production Architecture - Single-Pass Implementation Ready  
**Version:** 6.0 - Enterprise Grade with Complete Execution Logic  
**Last Updated:** 2025-01-20

---

## 📊 CURRENT STATE vs TARGET STATE

### **Current Worker: 3/10**

**Critical Failures:**
- ❌ Synchronous blocking (30s Worker timeout kills long analyses)
- ❌ Service role everywhere (RLS bypassed = security risk)
- ❌ Zero observability (no cost tracking, no performance metrics)
- ❌ No progress tracking (users refresh = lost state)
- ❌ No retry logic (Apify fails = user loses credits permanently)
- ❌ Mixed architecture (40+ scattered controller files)
- ❌ Race conditions (duplicate credit deductions possible)
- ❌ user_id instead of account_id (wrong multi-tenant model)
- ❌ No duplicate analysis prevention (spam clicks = double charge)

**What Works:**
- ✅ AWS Secrets integration
- ✅ Path aliases configured (Wrangler handles via esbuild automatically)
- ✅ Domain folders exist

---

### **Target Worker: 97/100**

**Core Infrastructure:**
- ✅ Workflows = async orchestration (no timeout limits)
- ✅ Dual Supabase client (RLS enforced + admin for system ops)
- ✅ Cloudflare AI Gateway (30-40% cost savings, FREE)
- ✅ Durable Objects (real-time progress + cancel capability)
- ✅ Feature-first architecture (vertical slices)
- ✅ Comprehensive observability (Analytics Engine + Sentry)
- ✅ Cloudflare Cron Triggers (monthly renewals, daily cleanup)

**Performance Gains:**
- LIGHT: 10-13s → 6s avg (with 50% cache hit rate)
- DEEP: 30s → 18-23s cold, 12-15s cached
- XRAY: 35s → 16-18s cold, 10-12s cached
- Bulk 100 profiles: 16min → 60-120s

**Cost Reductions:**
- AI costs: $300/mo → $210/mo (prompt caching + AI Gateway)
- Worker CPU: $100/mo → $2/mo (Workflows handle heavy lifting)
- Total savings: **$188/month**

**Why not 100/100?** Needs production battle-testing for final edge cases.

---

## 🎯 ARCHITECTURE OVERVIEW

### **Folder Structure**

```
src/
├── features/              # Vertical slices (domain-driven)
│   ├── credits/
│   │   ├── domain/        # Credit business rules, value objects
│   │   ├── application/   # Use cases (get balance, deduct credits)
│   │   ├── controllers/   # HTTP endpoints
│   │   └── routes.ts
│   ├── auth/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── controllers/
│   │   └── routes.ts
│   ├── business/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── controllers/
│   │   └── routes.ts
│   └── analysis/
│       ├── domain/
│       ├── application/
│       ├── controllers/
│       └── routes.ts
│
├── core/                  # Horizontal services
│   ├── workflows/
│   │   ├── deep-analysis.workflow.ts
│   │   ├── light-analysis.workflow.ts
│   │   ├── xray-analysis.workflow.ts
│   │   ├── bulk-analysis.workflow.ts
│   │   └── steps/
│   ├── queues/
│   │   ├── stripe-webhook.consumer.ts
│   │   └── analysis.consumer.ts
│   ├── durable-objects/
│   │   └── analysis-progress.do.ts
│   └── cron/
│       ├── monthly-renewal.job.ts
│       ├── daily-cleanup.job.ts
│       ├── failed-analysis-cleanup.job.ts
│       └── subscription-sync.job.ts (optional)
│
├── infrastructure/        # External adapters
│   ├── database/
│   │   ├── supabase.client.ts
│   │   └── repositories/
│   │       ├── base.repository.ts
│   │       ├── credits.repository.ts
│   │       ├── business.repository.ts
│   │       ├── leads.repository.ts
│   │       └── analysis.repository.ts
│   ├── ai/
│   │   ├── openai.adapter.ts
│   │   ├── claude.adapter.ts
│   │   └── ai-gateway.client.ts
│   ├── scraping/
│   │   └── apify.adapter.ts
│   ├── cache/
│   │   └── r2-cache.service.ts
│   └── monitoring/
│       ├── analytics-engine.client.ts
│       ├── sentry.client.ts
│       ├── cost-tracker.service.ts
│       └── performance-tracker.service.ts
│
└── shared/               # Pure utilities
    ├── middleware/
    │   ├── auth.middleware.ts
    │   ├── rate-limit.middleware.ts
    │   ├── error.middleware.ts
    │   └── analytics.middleware.ts
    ├── utils/
    │   ├── logger.util.ts
    │   ├── validation.util.ts
    │   └── response.util.ts
    └── types/
```

---

## 🔧 CRITICAL EXECUTION LOGIC

### **1. Analysis Execution Flow (Complete 12-Step Process)**

```
1. Receive request → Generate run_id
   ↓
2. Create pending record in runs table
   ↓
3. ✅ CHECK: Duplicate analysis already running?
   - Query: SELECT run_id FROM analyses 
     WHERE username = $1 AND account_id = $2 
     AND status IN ('pending', 'processing')
     AND created_at > NOW() - INTERVAL '5 minutes'
   - If exists → Return error: "Analysis already in progress"
   - If not exists → Continue
   ↓
4. ✅ CHECK: Sufficient credits?
   - Query credit_balances for account_id
   - If insufficient → Return error: "Insufficient credits"
   ↓
5. Deduct credits IMMEDIATELY (atomic RPC call)
   - Prevents race conditions
   - User committed to purchase
   - If later steps fail → Auto-refund
   ↓
6. Check R2 cache for profile
   - Key: instagram:${username}:v1
   - If cache hit AND not expired → Skip to step 9
   - If cache miss or expired → Continue to step 7
   ↓
7. Scrape profile via Apify (6-8s bottleneck)
   - With retry on infrastructure failures
   ↓
8. Store scraped profile in R2 cache
   - TTL: 24h (light), 12h (deep), 6h (xray)
   ↓
9. Run AI analysis (parallel calls for DEEP/XRAY)
   - Business context cached (prompt caching)
   - Full parallelization (Promise.all)
   ↓
10. Upsert lead (not insert)
   - Check existing: username + account_id + business_profile_id + deleted_at IS NULL
   - If exists → Update follower_count, bio, profile_pic_url, last_analyzed_at
   - If not exists → Insert new lead
   ↓
11. Save analysis results to database
   ↓
12. Update run status to 'complete'
    Update Durable Object progress to 100%
```

**Key Decision: Credit Deduction Timing**
- **BEFORE scraping** (Step 5) = RECOMMENDED
- ✅ Prevents race conditions (user can't spam multiple requests)
- ✅ User committed at point of request
- ❌ Must refund if later steps fail (acceptable trade-off)

---

### **2. R2 Cache Strategy**

**Cache Key Structure:**
```typescript
const cacheKey = `instagram:${username}:v1`;
// v1 = version (bump when scraping logic changes)
// v2 migration: Keep v1 alive 7 days, new analyses use v2
```

**TTL by Analysis Type:**
- LIGHT: 24 hours (profile changes less critical)
- DEEP: 12 hours (balanced freshness)
- XRAY: 6 hours (premium = fresher data)

**Cache Invalidation Triggers:**
- TTL expired (checked on retrieval)
- User manually refreshes profile (bypass cache flag)
- Follower count changed >10% since cached
- Profile bio changed significantly

**Cache Miss Handling:**
```typescript
if (!cached || isCacheExpired(cached, analysisType)) {
  // Scrape fresh data
  const profile = await scrapeProfile(username);
  // Store in cache
  await r2Cache.set(username, profile);
}
```

**Storage Limits:**
- Average profile: 50KB JSON
- 10,000 cached profiles: 500MB
- Cost: $0.0075/month (negligible)
- **Conclusion:** No storage limits needed

---

### **3. Prompt Caching Implementation**

**Business Context Structure (800 tokens - CACHED):**
```typescript
const businessContext = `
Company: ${business.company_name}
Industry: ${business.industry}
Target Audience: ${business.target_audience}

ICP Criteria:
- Follower Range: ${business.icp_min_followers} - ${business.icp_max_followers}
- Engagement Rate: >${business.icp_min_engagement_rate}%
- Content Themes: ${business.icp_content_themes.join(', ')}
- Geographic Focus: ${business.icp_geographic_focus}

Key Selling Points:
${business.selling_points.map((p, i) => `${i+1}. ${p}`).join('\n')}

Brand Voice: ${business.brand_voice}
Outreach Goals: ${business.outreach_goals}
`;
```

**Profile Data (NEVER cached - changes per request):**
```typescript
const profileData = `
Username: @${profile.username}
Followers: ${profile.followersCount}
Bio: ${profile.biography}
Recent Posts: [${profile.posts.length} posts analyzed]
Engagement Rate: ${profile.engagementRate}%
...
`;
```

**OpenAI Auto-Caching:**
- Place business context FIRST in messages array
- After 1024+ tokens, OpenAI auto-caches
- No special markers needed
- Cost: $0.003 → $0.0003 per cached analysis

**Claude Explicit Caching:**
```typescript
{
  role: 'user',
  content: [
    {
      type: 'text',
      text: businessContext,
      cache_control: { type: 'ephemeral' }  // Claude-specific marker
    },
    {
      type: 'text',
      text: profileData  // Not cached
    }
  ]
}
```

**Cache Hit Rate Optimization:**
- If business profile changes → Cache misses increase
- Solution: Include business_profile.updated_at in cache key
- OR: Hash business context for stable identifier

---

### **4. AI Model Selection Reasoning**

**LIGHT Analysis:**
- Model: `gpt-4o-mini`
- Reasoning: Fastest (2-3s), cheapest ($0.15/1M input), excellent at JSON structured output
- Reasoning Effort: `low` (simple scoring task)
- Why not gpt-5-mini: Overkill for basic scoring, 2x slower

**DEEP Analysis:**
- **Core Strategy:** `gpt-5-mini` (12-15s)
  - Reasoning: Needs deeper analysis than gpt-4o-mini can provide
  - Reasoning Effort: `medium` (balanced)
  - Why not gpt-5: Marginal quality gain, 2x cost
- **Outreach:** `gpt-5-mini` (3-4s)
  - Reasoning Effort: `low` (tactical, template-based)
- **Personality:** `gpt-5-mini` (3-4s)
  - Reasoning Effort: `low` (pattern recognition)

**XRAY Analysis:**
- **Psychographic:** `gpt-5` (7-10s)
  - Reasoning: Premium feature, users expect deepest insight
  - Reasoning Effort: `medium` (not `high` - adds 10-20s for only 5% quality gain)
  - Why worth it: Deepest psychological analysis justifies premium model
- **Commercial:** `gpt-5-mini` (5-6s)
  - Reasoning Effort: `medium` (strategic decisions)
- **Outreach:** `gpt-5-mini` (3-4s)
- **Personality:** `gpt-5-mini` (3-4s)

**Why Never 'high' Reasoning Effort:**
- Adds 10-20s per call
- Quality gain: ~5%
- Not worth latency impact on UX

---

### **5. Full AI Parallelization**

**BEFORE (Sequential - SLOW):**
```typescript
// DEEP: 19s total
const core = await executeCoreStrategy();      // 15s
const outreach = await executeOutreach();      // 4s (waits for core)
const personality = await executePersonality(); // 4s (waits for outreach)
```

**AFTER (Parallel - FAST):**
```typescript
// DEEP: 15s total (4s saved)
const [core, outreach, personality] = await Promise.all([
  executeCoreStrategy(),   // 15s
  executeOutreach(),       // 4s (parallel)
  executePersonality()     // 4s (parallel)
]);
// Total time = max(15, 4, 4) = 15s

// XRAY: 10s total (12s saved from 22s)
const [psycho, commercial, outreach, personality] = await Promise.all([
  executePsychographic(),  // 10s
  executeCommercial(),     // 6s
  executeOutreach(),       // 4s
  executePersonality()     // 4s
]);
// Total time = max(10, 6, 4, 4) = 10s
```

---

### **6. Bulk Analysis Apify-Aware Batching**

**Problem:** Apify Starter plan = 10 concurrent actors maximum

**Solution:**
```typescript
const APIFY_CONCURRENCY = 10;

// Split 100 usernames into 10 batches of 10
const batches = chunkArray(usernames, APIFY_CONCURRENCY);
// [[1-10], [11-20], [21-30], ..., [91-100]]

// Process sequentially (each batch waits for previous)
for (const batch of batches) {
  await Promise.all(batch.map(username => scrapeProfile(username)));
  // Batch 1: 10 profiles, 8s
  // Batch 2: 10 profiles, 8s
  // ...
  // Total: 10 batches × 8s = 80s
}
```

**Partial Failure Handling:**
```
Batch 1: 10 profiles
- 9 succeed
- 1 fails (Apify timeout)

Action: Retry failed profile 3x with exponential backoff (1s, 2s, 4s)
If all 3 retries fail:
  - Mark analysis as failed
  - Refund 1 credit via deduct_credits(-1)
  - Continue processing remaining profiles
  - Don't fail entire batch
```

**Progress Granularity:**
```typescript
{
  total: 100,
  completed: 47,
  succeeded: 45,
  failed: 2,
  current_batch: 5,
  total_batches: 10,
  batch_progress: "Processing batch 5/10 (profiles 41-50)",
  eta_seconds: 53
}
```

---

### **7. Cost Tracking Formulas**

**Apify Cost Calculation:**
```typescript
// Apify charges $0.25 per compute unit (CU)
// 1 CU = 1 hour of scraping
// Instagram scraper: ~0.01 CU per profile (36s avg)

function calculateApifyCost(durationMs: number): number {
  const computeUnits = durationMs / (1000 * 60 * 60); // ms to hours
  const cost = computeUnits * 0.25;
  return cost;
}

// Example: 50 posts, 8 seconds
// = 8000ms / 3600000 = 0.00222 CU
// = 0.00222 × $0.25 = $0.00056
```

**AI Cost Per Model:**
```typescript
const AI_PRICING = {
  'gpt-4o-mini': {
    per_1m_input: 0.15,
    per_1m_output: 0.60
  },
  'gpt-5-mini': {
    per_1m_input: 0.30,
    per_1m_output: 1.20
  },
  'gpt-5': {
    per_1m_input: 0.60,
    per_1m_output: 2.40
  },
  'claude-3-5-sonnet': {
    per_1m_input: 3.00,
    per_1m_output: 15.00
  }
};

function calculateAICost(model: string, tokensIn: number, tokensOut: number): number {
  const pricing = AI_PRICING[model];
  const inputCost = (tokensIn / 1_000_000) * pricing.per_1m_input;
  const outputCost = (tokensOut / 1_000_000) * pricing.per_1m_output;
  return inputCost + outputCost;
}
```

**Margin Calculation:**
```
LIGHT: 1 credit = $0.97 revenue
Costs: Apify ($0.0006) + AI ($0.0003) = $0.0009
Margin: $0.97 - $0.0009 = $0.9691 (99.9%)

DEEP: 5 credits = $4.85 revenue
Costs: Apify ($0.0006) + AI ($0.015) = $0.0156
Margin: $4.85 - $0.0156 = $4.8344 (99.7%)

XRAY: 6 credits = $5.82 revenue
Costs: Apify ($0.0006) + AI ($0.025) = $0.0256
Margin: $5.82 - $0.0256 = $5.7944 (99.6%)
```

---

### **8. Durable Object Lifecycle**

**Creation → Active → Cleanup Flow:**

```
1. CREATION (analysis starts)
   POST /v1/analyze
   ↓
   Generate run_id
   ↓
   Create DO: ANALYSIS_PROGRESS.idFromName(run_id)
   ↓
   Initialize state: { run_id, status: 'queued', progress: 0 }

2. ACTIVE PHASE (during analysis, 10-30s)
   Workflow updates DO every step:
   - "Checking cache" (10%)
   - "Scraping profile" (30%)
   - "Running AI analysis" (70%)
   - "Saving results" (90%)
   - "Complete" (100%)
   
   Client polls DO every 2s:
   GET /runs/:run_id/progress

3. COMPLETION
   Analysis finishes
   ↓
   Workflow sends final update: { status: 'complete', progress: 100 }
   ↓
   DO sets cleanup alarm: ctx.storage.setAlarm(Date.now() + 3600000) // 1 hour
   ↓
   Client stops polling (sees status: 'complete')

4. CLEANUP (1 hour after completion)
   Alarm fires
   ↓
   DO deletes its own state: ctx.storage.deleteAll()
   ↓
   DO becomes inactive (Cloudflare garbage collects)
```

**Hibernation Decision:**
- **Disabled** for OSLIRA analyses (all complete in <30s)
- Hibernation beneficial only if DO idle >10s between updates
- LIGHT/DEEP/XRAY too fast to benefit
- Keeping DO always active = simpler, negligible cost difference

**Fallback Strategy (DO Unavailable):**
```typescript
async function getAnalysisProgress(run_id: string, env: Env) {
  try {
    // Try DO first (primary path)
    const doId = env.ANALYSIS_PROGRESS.idFromName(run_id);
    const progress = await doId.fetch('/progress');
    return await progress.json();
  } catch (error) {
    // Graceful degradation: Fall back to database
    const run = await db.query(
      'SELECT status, created_at FROM runs WHERE run_id = $1',
      [run_id]
    );
    
    return {
      run_id,
      status: run.status,
      progress: run.status === 'complete' ? 100 : 50,
      current_step: 'Processing...',
      fallback: true,  // Flag indicates degraded mode
      source: 'database'
    };
  }
}
```

**Cost Analysis:**
- DO pricing: $0.15 per million requests
- Typical analysis:
  - 1 DO create
  - 10-20 progress updates (workflow → DO)
  - 10-15 progress polls (client → DO)
  - 4-6 cancel checks (workflow checks between steps)
  - Total: ~30-50 DO requests
  - Cost: $0.0000075 per analysis (negligible)

---

### **9. Webhook Idempotency (Race Condition Handling)**

**Problem:**
```
Request A: Check webhook_events → Not found → Insert
Request B: Check webhook_events → Not found → Insert
  ↓
Both think they're first
  ↓
Process twice
  ↓
Double credit grant
```

**Solution: ON CONFLICT DO NOTHING**
```sql
INSERT INTO webhook_events (stripe_event_id, event_type, received_at)
VALUES ($1, $2, NOW())
ON CONFLICT (stripe_event_id) DO NOTHING
RETURNING id;

-- If RETURNING id IS NULL → Another request already inserted (lost race)
-- If RETURNING id HAS value → We won race, proceed with processing
```

**Complete Flow:**
```typescript
export async function handleStripeWebhook(request: Request, env: Env) {
  // 1. Verify signature (BEFORE any database operations)
  const signature = request.headers.get('stripe-signature');
  const payload = await request.text();
  
  let event;
  try {
    event = stripe.webhooks.constructEvent(payload, signature, webhookSecret);
  } catch (err) {
    return new Response('Invalid signature', { status: 400 });
  }
  
  // 2. Idempotency check (prevents race conditions)
  const result = await db.query(`
    INSERT INTO webhook_events (stripe_event_id, event_type, received_at)
    VALUES ($1, $2, NOW())
    ON CONFLICT (stripe_event_id) DO NOTHING
    RETURNING id
  `, [event.id, event.type]);
  
  if (!result.rows[0]) {
    // Already processed by another request (duplicate delivery)
    console.log(`Webhook ${event.id} already processed`);
    return new Response('OK', { status: 200 }); // Must return 200 to Stripe
  }
  
  // 3. Queue for async processing (don't block Stripe)
  await env.STRIPE_WEBHOOKS_QUEUE.send({
    event_id: event.id,
    event_type: event.type,
    event_data: event.data
  });
  
  // 4. Return 200 immediately (Stripe expects response <5s)
  return new Response('OK', { status: 200 });
}
```

---

### **10. Lead Upsert Logic**

**Field Update Priority:**

**Always Update:**
- `follower_count` (changes frequently)
- `bio` (users change often)
- `profile_pic_url` (profile photos change)
- `last_analyzed_at` (timestamp of most recent analysis)
- `is_verified` (status can change)

**Never Update:**
- `first_analyzed_at` (historical record, set once)
- `created_at` (audit trail)
- `account_id`, `business_profile_id` (foreign keys, immutable)

**Conditionally Update:**
- `instagram_url` → Only if currently NULL (URL doesn't change)

**SQL Implementation:**
```sql
INSERT INTO leads (
  username, account_id, business_profile_id,
  follower_count, bio, profile_pic_url, instagram_url,
  is_verified, first_analyzed_at, last_analyzed_at
)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW(), NOW())
ON CONFLICT (username, account_id, business_profile_id)
WHERE deleted_at IS NULL
DO UPDATE SET
  follower_count = EXCLUDED.follower_count,
  bio = EXCLUDED.bio,
  profile_pic_url = EXCLUDED.profile_pic_url,
  instagram_url = COALESCE(leads.instagram_url, EXCLUDED.instagram_url),
  is_verified = EXCLUDED.is_verified,
  last_analyzed_at = NOW()
RETURNING id, (xmax = 0) AS is_new;

-- xmax = 0 means INSERT happened (new lead)
-- xmax != 0 means UPDATE happened (existing lead)
```

**Repository Method Returns:**
```typescript
interface UpsertLeadResult {
  leadId: string;
  isNew: boolean;  // true = newly created, false = updated existing
}
```

---

### **11. Cron Job Idempotency**

**Monthly Renewal Deduplication:**

**Problem:** Cron runs twice (Cloudflare retry) → Double credit grant

**Solution: Check credit_ledger for recent grant**
```sql
-- BEFORE granting credits, check:
SELECT id FROM credit_ledger
WHERE account_id = $1
  AND operation_type = 'subscription_renewal'
  AND created_at > NOW() - INTERVAL '23 hours'
LIMIT 1;

-- If exists → Skip (already renewed in past 23 hours)
-- If not exists → Proceed with credit grant via deduct_credits()
```

**Why 23 hours not 24:**
- Cron scheduled for 3 AM UTC
- 23-hour window allows for clock drift
- Prevents duplicate if cron runs slightly early next day

**Failed Analysis Timeout Logic:**

**Decision: 5-minute timeout threshold**
- LIGHT expected: 11s max, timeout after 5min = 27x safety margin
- DEEP expected: 23s max, timeout after 5min = 13x safety margin
- XRAY expected: 18s max, timeout after 5min = 16x safety margin

**SQL + Refund:**
```sql
-- Find stuck analyses
UPDATE analyses
SET status = 'failed',
    error_message = 'Analysis timed out after 5 minutes'
WHERE status IN ('pending', 'processing')
  AND created_at < NOW() - INTERVAL '5 minutes'
RETURNING id, account_id, credits_used;

-- For each returned analysis, refund credits
CALL deduct_credits(
  analysis.account_id,
  -analysis.credits_used,  -- Negative amount = refund
  'timeout_refund',
  'Auto-refund for timed out analysis'
);
```

---

### **12. RLS Performance Fix Impact**

**BEFORE (Slow - 450ms):**
```sql
CREATE POLICY "Users see own leads"
ON leads FOR SELECT
USING (account_id = auth.uid());

-- Query plan:
-- Seq Scan on leads (cost=0.00..1234.56)
--   Filter: (account_id = auth.uid())
-- Execution: 450ms (auth.uid() called per row scanned - 10,000 times)
```

**AFTER (Fast - 4ms):**
```sql
CREATE POLICY "Users see own leads"
ON leads FOR SELECT
USING (account_id = (SELECT auth.uid()));

-- Query plan:
-- Index Scan using leads_account_id_idx (cost=0.29..8.31)
--   Index Cond: (account_id = $1)
-- Execution: 4ms (auth.uid() called once, result cached in query plan)
```

**Why 100x Faster:**
- Without subquery: auth.uid() re-executes for every row scanned
- With subquery: auth.uid() executes once, result reused
- Database uses index seek instead of full table scan

**Apply to ALL these tables:**
- ✅ leads (largest table, most queries)
- ✅ analyses (high query frequency)
- ✅ business_profiles
- ✅ credit_balances
- ✅ credit_transactions
- ✅ subscriptions
- ❌ accounts (no RLS, admin-only access)

---

### **13. Async Execution UX Pattern**

**Decision: HTTP Polling over WebSocket**

**Why Polling:**
- ✅ Simple (standard REST, no connection management)
- ✅ Scales well (stateless requests)
- ✅ 2-second delay acceptable for 10-30s analyses
- ✅ No reconnection logic needed

**Why Not WebSocket:**
- ❌ Complex (connection lifecycle, reconnects, heartbeats)
- ❌ Overkill for short analyses (not real-time chat)
- ❌ More expensive (keep-alive connections)

**Frontend Implementation:**
```typescript
async function startAnalysis(username: string) {
  // 1. Initiate analysis (returns immediately)
  const response = await fetch('/v1/analyze', {
    method: 'POST',
    body: JSON.stringify({ 
      username, 
      analysis_type: 'deep',
      business_id: 'biz_123' 
    })
  });
  
  const { run_id, estimated_time } = await response.json();
  // { run_id: 'run_abc123', status: 'queued', estimated_time: '18-23s' }
  
  // 2. Poll for progress every 2 seconds
  const pollInterval = setInterval(async () => {
    const progress = await fetch(`/runs/${run_id}/progress`);
    const data = await progress.json();
    
    // Update UI
    updateProgressBar(data.progress);
    updateStatusText(data.current_step);
    updateElapsedTime(data.elapsed_ms);
    
    // Stop polling when done
    if (['complete', 'failed', 'cancelled'].includes(data.status)) {
      clearInterval(pollInterval);
      
      if (data.status === 'complete') {
        showResults(data.result);
      } else if (data.status === 'failed') {
        showError(data.error_message);
      } else {
        showCancelled();
      }
    }
  }, 2000);
  
  // 3. Cleanup on component unmount
  return () => clearInterval(pollInterval);
}
```

**Cancel Button:**
```typescript
cancelButton.onclick = async () => {
  await fetch(`/runs/${run_id}/cancel`, { method: 'POST' });
  // DO sets should_cancel flag
  // Workflow checks flag between steps
  // Analysis stops gracefully, doesn't waste credits/API calls
};
```

---

## 🎯 CRITICAL DECISIONS LOCKED

### **1. Dual Supabase Client Pattern**
**Status:** MANDATORY

**Use Cases:**
- **User Client (anon key):** All SELECT queries, user-facing operations, RLS enforced
- **Admin Client (service role):** System operations (cron jobs, workflows), credit deductions, RLS bypassed

**Factory Pattern:**
```typescript
export class SupabaseClientFactory {
  createUserClient(userId: string): SupabaseClient {
    return createClient(supabaseUrl, anonKey);
  }
  
  createAdminClient(): SupabaseClient {
    return createClient(supabaseUrl, serviceRoleKey);
  }
}
```

---

### **2. AI Gateway Integration**
**Status:** MANDATORY (FREE, 30-40% savings)

**One-Line Change:**
```typescript
// Before
const openai = new OpenAI({
  baseURL: "https://api.openai.com/v1"
});

// After
const openai = new OpenAI({
  baseURL: "https://gateway.ai.cloudflare.com/v1/{account_id}/{gateway_name}/openai"
});
```

**Benefits:**
- 30-40% cost reduction (caches duplicate prompts across users)
- Automatic fallback (OpenAI down → Claude)
- Free analytics (track costs per user, per endpoint)
- Zero additional cost, zero downside

---

### **3. Workflow Error Handling Philosophy**
**Status:** LOCKED

**Retry infrastructure failures, NOT business logic failures:**

```typescript
// ✅ RETRY: Infrastructure failures (Apify timeout, OpenAI 503, network errors)
const profile = await step.do('scrape', async () => {
  return await scrapeProfile(username);
  // Cloudflare auto-retries on infrastructure failure
});

// ❌ NO RETRY: Business logic failures
const credits = await step.do('check_credits', async () => {
  if (userCredits < analysisConfig.credit_cost) {
    throw new BusinessLogicError('Insufficient credits');
  }
});
// Workflow stops immediately, no retry, clear error to user

// ✅ RETRY: External API failures
const analysis = await step.do('analyze', async () => {
  return await analyzeWithAI(profile);
  // Retries if OpenAI 503, rate limit, timeout
});
```

---

### **4. Secret Rotation**
**Status:** Accept 5-minute cache delay

**AWS Secrets Caching:**
```typescript
const secretsCache = new Map<string, { value: string; cachedAt: number }>();
const CACHE_TTL = 5 * 60 * 1000; // 5 minutes

// First request (cold start): ~200-500ms AWS fetch
// Subsequent requests (same Worker instance): <1ms cache hit
// Cache scope: Per Worker instance (not shared)
```

**Reasoning:** Secrets rarely rotate (monthly/quarterly). 5-minute delay acceptable for simplicity.

---

### **5. CI/CD**
**Status:** Manual deployment only

**Workflow:**
```bash
npm test                # Vitest suite
npm run typecheck       # TypeScript validation
npm run deploy          # wrangler deploy (manual trigger)
```

**Why No GitHub Actions:**
- Team size supports manual control
- Staging environment available for testing
- Deploy-on-demand matches workflow

---

### **6. Cron Jobs**
**Status:** Cloudflare Cron Triggers (NOT Supabase pg_cron)

**Why Cloudflare:**
- Already part of Worker architecture
- Better observability (Analytics Engine + Sentry)
- Easy testing via manual trigger endpoints
- Can call external APIs (Stripe, etc.)
- Consistent with rest of stack

**wrangler.toml:**
```toml
[triggers]
crons = [
  "0 3 1 * *",   # Monthly renewal (1st, 3 AM UTC)
  "0 2 * * *",   # Daily cleanup (2 AM UTC)
  "0 * * * *"    # Hourly failed analysis cleanup
]
```

---

## 📊 PERFORMANCE TARGETS

| Metric | Current | Target (Phase 1) | Target (Phase 3) |
|--------|---------|------------------|------------------|
| **LIGHT (cold)** | 10-13s | 8-11s | 6-9s |
| **LIGHT (cached)** | N/A | 2-3s | <1s |
| **DEEP (cold)** | 30s | 19-23s | 16-20s |
| **DEEP (cached)** | N/A | 12-15s | 10-12s |
| **XRAY (cold)** | 35s | 18-22s | 14-18s |
| **XRAY (cached)** | N/A | 10-12s | 8-10s |
| **Cost per LIGHT** | $0.0024 | $0.0005 | $0.0002 |
| **Bulk 100 profiles** | 16min | 80-120s | 60-90s |
| **Cache hit rate** | 0% | 30% | 60% |

**Key Constraint:** Apify 6-8s minimum (unavoidable bottleneck)

---

## 💰 COST BREAKDOWN

**Current Monthly Costs:**
- AI calls: $300/mo
- Worker CPU: $100/mo
- **Total:** $400/mo

**After Optimizations:**
- AI calls: $210/mo (30% reduction via AI Gateway + prompt caching)
- Worker CPU: $2/mo (98% reduction via Workflows)
- **Total:** $212/mo
- **Savings:** $188/month (47% reduction)

---

## ❌ FINALIZED - WILL NOT IMPLEMENT

1. **Gradual Deployments** - Staging available, all-at-once fine
2. **D1 for Caching** - R2 + KV sufficient
3. **Service Bindings** - Monolith with clear domain separation works
4. **Workers AI** - OpenAI/Claude already contracted
5. **Browser Rendering API** - Not in roadmap
6. **AutoRAG/Vectorize** - "Similar influencers" not in MVP
7. **Hyperdrive** - Supabase not bottleneck
8. **Supabase DB Webhooks** - Workflows notify directly

---

## 🔮 DEFERRED TO FUTURE

**Apify Fallback (Month 2+):**
- Trigger: Apify downtime >1 hour/month OR costs >30% budget
- Add: Bright Data adapter, ScraperAPI adapter
- Automatic failover on errors

**Advanced Caching (Month 3+):**
- Semantic caching for XRAY (embeddings-based, 90% similarity threshold)
- Batch API for multiple AI calls (50% discount, requires 24h processing delay)

**R2 Cache Expiration Cron (Month 2+):**
- Daily cleanup of expired cached profiles
- Storage cost optimization

---

# 📋 IMPLEMENTATION PHASES - REORDERED

## Phase 0: Foundation & Infrastructure Setup (Week 1)

### **Deliverables:**

**Project Bootstrap:**
- Initialize new repository with proper structure
- Configure `package.json`, `tsconfig.json`, `wrangler.toml`
- Install all dependencies
- Set up path aliases (@features, @core, @infrastructure, @shared)

**Cloudflare Bindings Validation:**
- Validate KV namespace accessible
- Validate R2 bucket accessible
- Validate Workflows binding configured
- Validate Queues binding configured
- Validate Durable Objects binding configured
- Validate Analytics Engine binding configured
- Health check endpoints with binding status

**AWS Secrets Integration:**
- Implement secrets caching (5-minute TTL)
- Test AWS Secrets Manager connection
- Verify all required secrets exist:
  - SUPABASE_URL
  - SUPABASE_ANON_KEY
  - SUPABASE_SERVICE_ROLE_KEY
  - APIFY_API_TOKEN
  - OPENAI_API_KEY
  - ANTHROPIC_API_KEY
  - STRIPE_SECRET_KEY
  - STRIPE_WEBHOOK_SECRET
  - SENTRY_DSN

**Database Verification:**
- Apply RLS fix to Supabase (wrap all `auth.uid()` calls in subqueries)
- Verify `SECURITY DEFINER` on all RPC functions:
  - `deduct_credits()`
  - `get_renewable_subscriptions()`
  - `generate_slug()`
- Test RLS performance improvement (450ms → 4ms)
- Verify indexes exist on critical columns

### **Success Criteria:**
- ✅ Worker boots without errors
- ✅ All bindings accessible and returning data
- ✅ Health check returns 200 with binding status
- ✅ AWS Secrets fetch working with caching
- ✅ Database queries 100x faster (RLS fix applied)
- ✅ All RPC functions have SECURITY DEFINER

---

## Phase 1: Core Infrastructure Layer (Week 2)

### **Build Order:**

```
infrastructure/
├── config/
│   └── secrets.ts                  # AWS Secrets Manager with caching
├── database/
│   ├── supabase.client.ts          # Dual client factory (anon + service role)
│   └── repositories/
│       ├── base.repository.ts      # Abstract base with common methods
│       ├── credits.repository.ts   # Credit operations
│       ├── business.repository.ts  # Business profile CRUD
│       ├── leads.repository.ts     # Lead upsert + queries
│       └── analysis.repository.ts  # Analysis CRUD + status updates
├── ai/
│   ├── ai-gateway.client.ts        # Cloudflare AI Gateway wrapper
│   ├── openai.adapter.ts           # OpenAI API calls with cost tracking
│   ├── claude.adapter.ts           # Anthropic API calls with cost tracking
│   └── pricing.config.ts           # AI model pricing constants
├── scraping/
│   └── apify.adapter.ts            # Apify Instagram scraper
├── cache/
│   └── r2-cache.service.ts         # Profile caching with TTL
└── monitoring/
    ├── analytics-engine.client.ts  # Cloudflare Analytics Engine
    ├── sentry.client.ts            # Error tracking (optional)
    ├── cost-tracker.service.ts     # Track Apify + AI costs per request
    └── performance-tracker.service.ts # Track step durations
```

### **Key Implementations:**

**Dual Supabase Client Factory:**
- `createUserClient()` - Uses anon key, RLS enforced, for frontend-like queries
- `createAdminClient()` - Uses service role, bypasses RLS, for system operations

**Repository Pattern:**
- Base repository with common CRUD operations
- Type-safe query builders
- Automatic error handling
- Soft delete support (checks `deleted_at IS NULL`)

**R2 Cache Service:**
- Key structure: `instagram:${username}:v1`
- TTL by analysis type (24h light, 12h deep, 6h xray)
- Cache invalidation logic
- Follower count change detection (>10% = invalidate)

**AI Gateway Integration:**
- Single configuration change for OpenAI/Claude
- Automatic 30-40% cost savings
- Fallback logic (OpenAI down → Claude)
- Request/response logging

**Cost & Performance Tracking:**
- Track Apify duration and cost per scrape
- Track AI tokens (input/output) and cost per call
- Track step durations (identify bottlenecks)
- Calculate profit margins per analysis

### **Success Criteria:**
- ✅ Can fetch secrets from AWS with caching
- ✅ Can query database with both anon and service role clients
- ✅ Can store and retrieve profiles from R2
- ✅ Can make AI calls through AI Gateway
- ✅ Can scrape Instagram profiles via Apify
- ✅ Cost tracking captures all expenses per request

---

## Phase 2: Shared Utilities Layer (Week 2-3)

### **Build Order:**

```
shared/
├── middleware/
│   ├── auth.middleware.ts          # JWT extraction and validation
│   ├── rate-limit.middleware.ts    # KV-based rate limiting
│   ├── error.middleware.ts         # Global error handler with Sentry
│   └── analytics.middleware.ts     # Track every request to Analytics Engine
├── utils/
│   ├── logger.util.ts              # Structured logging with levels
│   ├── validation.util.ts          # Zod schema validators
│   ├── response.util.ts            # Standard API response format
│   └── id-generator.util.ts        # Generate run_id, lead_id, etc.
├── types/
│   ├── env.types.ts                # Cloudflare Worker Env interface
│   ├── api.types.ts                # Request/response types
│   ├── analysis.types.ts           # Analysis domain types
│   └── database.types.ts           # Database table types
└── constants/
    ├── analysis.constants.ts       # Credit costs, model configs
    └── errors.constants.ts         # Standard error messages
```

### **Key Implementations:**

**Auth Middleware:**
- Extract JWT from Authorization header
- Validate token with Supabase
- Attach userId and accountId to context
- Return 401 for invalid/expired tokens

**Rate Limiting:**
- Store request counts in KV
- Key: `ratelimit:${userId}:${endpoint}`
- Configurable limits per endpoint (e.g., 100 analyses/hour)
- Return 429 with retry-after header

**Error Middleware:**
- Catch all unhandled errors
- Classify errors (client 4xx vs server 5xx)
- Log to Sentry (if configured)
- Return standardized error response
- Never expose internal details to client

**Standard Response Format:**
```typescript
{
  success: boolean,
  data?: T,
  error?: string,
  requestId: string,
  timestamp: string
}
```

**Logger Utility:**
- Structured JSON logging
- Log levels: debug, info, warn, error
- Include requestId in all logs
- Performance-friendly (no blocking I/O)

### **Success Criteria:**
- ✅ Auth middleware correctly validates JWT
- ✅ Rate limiting blocks excessive requests
- ✅ Error middleware catches and formats all errors
- ✅ Logger outputs structured JSON
- ✅ All responses follow standard format

---

## Phase 3: Domain Models & Configuration (Week 3)

### **Build Order:**

```
features/analysis/domain/
├── config/
│   └── analysis.config.ts          # LIGHT, DEEP, XRAY configurations
├── entities/
│   ├── analysis.entity.ts          # Analysis domain entity
│   └── lead.entity.ts              # Lead domain entity
├── value-objects/
│   ├── analysis-score.vo.ts        # Score validation (0-100)
│   ├── instagram-handle.vo.ts      # Username validation
│   └── analysis-type.vo.ts         # LIGHT/DEEP/XRAY enum
└── services/
    ├── lead-scorer.service.ts      # Score calculation logic
    └── duplicate-checker.service.ts # Check for in-progress analyses
```

### **Analysis Configuration (Centralized):**

```typescript
export const ANALYSIS_CONFIG = {
  LIGHT: {
    credit_cost: 1,
    price_usd: 0.97,
    ai_model: 'gpt-4o-mini',
    reasoning_effort: 'low',
    cache_ttl_hours: 24,
    posts_limit: 12
  },
  DEEP: {
    credit_cost: 5,
    price_usd: 4.85,
    ai_models: {
      core_strategy: 'gpt-5-mini',
      outreach: 'gpt-5-mini',
      personality: 'gpt-5-mini'
    },
    cache_ttl_hours: 12,
    posts_limit: 50
  },
  XRAY: {
    credit_cost: 6,
    price_usd: 5.82,
    ai_models: {
      psychographic: 'gpt-5',
      commercial: 'gpt-5-mini',
      outreach: 'gpt-5-mini',
      personality: 'gpt-5-mini'
    },
    cache_ttl_hours: 6,
    posts_limit: 50
  }
} as const;
```

### **Domain Entities:**

**Analysis Entity:**
- Validation rules (score 0-100, valid status enum)
- Business logic (canBeRefunded, isComplete, isFailed)
- Immutability (can't change completed analysis)

**Lead Entity:**
- Instagram username validation
- Follower count validation (>0)
- Upsert logic (update if exists, insert if new)

**Value Objects:**
- Immutable, validated primitives
- Self-documenting (e.g., InstagramHandle vs string)
- Type safety at compile time

### **Success Criteria:**
- ✅ Analysis config centralized and type-safe
- ✅ Domain entities enforce business rules
- ✅ Value objects prevent invalid data
- ✅ No business logic leaks into controllers

---

## Phase 4: Credits Feature (Week 4)

### **Build Order:**

```
features/credits/
├── domain/
│   ├── credit-amount.vo.ts         # Positive/negative amount validation
│   └── credit-calculator.service.ts # Calculate costs, refunds
├── application/
│   ├── get-balance.usecase.ts      # Fetch current balance
│   ├── get-transactions.usecase.ts # Fetch transaction history
│   └── ports/
│       └── credit.repository.interface.ts # Repository contract
├── controllers/
│   ├── balance.controller.ts       # GET /credits/balance
│   └── transactions.controller.ts  # GET /credits/transactions
└── routes.ts                        # Register credit routes
```

### **Key Implementations:**

**Get Balance Use Case:**
- Query `credit_balances` table (fast)
- Filter by accountId from JWT
- Return current balance + metadata

**Get Transactions Use Case:**
- Query `credit_ledger` table with pagination
- Filter by accountId from JWT
- Order by created_at DESC
- Include related user (who triggered transaction)

**Credit Deduction (Backend Only):**
- NEVER called from frontend directly
- Always use `deduct_credits()` RPC function
- Handled by Worker with service role key
- Atomic operation (locks row, validates balance)

### **API Endpoints:**

```typescript
GET /credits/balance
Response: {
  success: true,
  data: {
    current_balance: 48,
    account_id: 'acc_123'
  }
}

GET /credits/transactions?limit=50&offset=0
Response: {
  success: true,
  data: {
    transactions: [
      {
        id: 'tx_123',
        amount: -2,
        balance_after: 48,
        transaction_type: 'analysis',
        description: 'Deep analysis of @nike',
        created_at: '2025-01-15T10:30:00Z'
      }
    ],
    total_count: 247,
    has_more: true
  }
}
```

### **Success Criteria:**
- ✅ Balance endpoint returns correct balance
- ✅ Transactions endpoint paginated correctly
- ✅ RLS ensures users only see their account's data
- ✅ Credits feature validates architecture pattern

---

## Phase 5: Auth Feature (Week 4)

### **Build Order:**

```
features/auth/
├── domain/
│   └── jwt-validator.service.ts    # Validate JWT structure
├── application/
│   ├── verify-token.usecase.ts     # Verify with Supabase
│   └── ports/
│       └── auth.repository.interface.ts
├── controllers/
│   └── auth.controller.ts          # GET /auth/verify
└── routes.ts
```

### **Key Implementations:**

**JWT Validation:**
- Extract token from Authorization header
- Verify signature with Supabase
- Decode payload (userId, accountId, role)
- Return 401 if invalid/expired

**Token Verification Use Case:**
- Call Supabase auth.getUser()
- Return user profile + account memberships
- Cache result for 5 minutes (reduce Supabase calls)

### **API Endpoint:**

```typescript
GET /auth/verify
Headers: { Authorization: 'Bearer <jwt>' }
Response: {
  success: true,
  data: {
    user_id: 'user_123',
    email: 'hamza@example.com',
    accounts: [
      {
        account_id: 'acc_123',
        role: 'owner',
        credits: 48
      }
    ]
  }
}
```

### **Success Criteria:**
- ✅ Valid JWT returns user data
- ✅ Invalid JWT returns 401
- ✅ Expired JWT returns 401
- ✅ Middleware reuses auth logic

---

## Phase 6: Business Profiles Feature (Week 5)

### **Build Order:**

```
features/business/
├── domain/
│   ├── business-profile.entity.ts  # Business profile domain model
│   ├── icp-matcher.service.ts      # Match lead to ICP criteria
│   └── value-objects/
│       └── icp-criteria.vo.ts      # Follower range, engagement, etc.
├── application/
│   ├── use-cases/
│   │   ├── create-profile.usecase.ts
│   │   ├── update-profile.usecase.ts
│   │   ├── get-profiles.usecase.ts
│   │   └── delete-profile.usecase.ts
│   └── ports/
│       └── business.repository.interface.ts
├── controllers/
│   └── profiles.controller.ts      # CRUD endpoints
└── routes.ts
```

### **Key Implementations:**

**ICP Criteria:**
- Follower range (min/max)
- Engagement rate minimum
- Content themes (array of strings)
- Geographic focus
- Industry/niche

**Business Profile Entity:**
- Validation (required fields)
- Default values (brand_voice, outreach_goals)
- Relationship to account (many profiles per account)

**CRUD Operations:**
- Create: Validate ICP criteria, generate slug
- Read: List all profiles for account, get single profile
- Update: Partial updates, preserve audit trail
- Delete: Soft delete (set deleted_at)

### **API Endpoints:**

```typescript
GET /business-profiles
POST /business-profiles
GET /business-profiles/:id
PUT /business-profiles/:id
DELETE /business-profiles/:id
```

### **Success Criteria:**
- ✅ Can create business profile with ICP
- ✅ Can list all profiles for account
- ✅ Can update profile without breaking relationships
- ✅ Soft delete preserves data integrity

---

## Phase 7: Analysis Feature - Core Product (Weeks 5-6)

### **Build Order:**

```
features/analysis/
├── domain/
│   ├── entities/ (from Phase 3)
│   ├── value-objects/ (from Phase 3)
│   ├── services/
│   │   ├── ai-analysis.service.ts      # All AI calls (LIGHT/DEEP/XRAY)
│   │   ├── prompt-builder.service.ts   # Build prompts with caching
│   │   └── result-parser.service.ts    # Parse JSON responses
│   └── config/ (from Phase 3)
├── application/
│   ├── use-cases/
│   │   ├── analyze-lead.usecase.ts     # Main analysis orchestration
│   │   ├── bulk-analyze.usecase.ts     # Batch analysis
│   │   ├── anonymous-analyze.usecase.ts # No auth required
│   │   ├── get-analysis.usecase.ts     # Fetch results
│   │   └── cancel-analysis.usecase.ts  # Cancel in-progress
│   ├── ports/
│   │   ├── ai-provider.interface.ts    # AI adapter contract
│   │   └── scraper.interface.ts        # Scraper adapter contract
│   └── dtos/
│       └── analysis.dto.ts             # Request/response DTOs
├── controllers/
│   ├── analyze.controller.ts           # POST /v1/analyze
│   ├── bulk-analyze.controller.ts      # POST /v1/bulk-analyze
│   ├── anonymous-analyze.controller.ts # POST /v1/analyze-anonymous
│   ├── results.controller.ts           # GET /runs/:run_id
│   └── progress.controller.ts          # GET /runs/:run_id/progress
└── routes.ts
```

### **Key Implementations:**

**Analyze Lead Use Case (12-Step Flow):**
1. Generate run_id
2. Create pending record in `analyses` table
3. Check duplicate analysis in progress (5min window)
4. Check sufficient credits
5. Deduct credits (atomic RPC call)
6. Check R2 cache for profile
7. Scrape profile via Apify (if cache miss)
8. Store in R2 cache
9. Run AI analysis (parallel calls for DEEP/XRAY)
10. Upsert lead (update if exists, insert if new)
11. Save analysis results to database
12. Update status to 'complete'

**AI Analysis Service:**
- Build business context (800 tokens, cached)
- Build profile data (dynamic, not cached)
- Call appropriate model based on analysis type
- Parse structured JSON response
- Handle errors and retries

**Prompt Builder:**
- Separate cached vs non-cached sections
- OpenAI auto-caches after 1024 tokens
- Claude requires explicit cache_control markers
- Include business context first for caching

**Bulk Analysis:**
- Batch usernames into groups of 10 (Apify limit)
- Process batches sequentially
- Track progress (completed/failed counts)
- Refund credits on individual failures

### **API Endpoints:**

```typescript
POST /v1/analyze
Body: {
  username: 'nike',
  business_id: 'biz_123',
  analysis_type: 'deep'
}
Response: {
  success: true,
  data: {
    run_id: 'run_abc123',
    status: 'queued',
    estimated_time: '18-23s'
  }
}

GET /runs/:run_id/progress
Response: {
  success: true,
  data: {
    run_id: 'run_abc123',
    status: 'analyzing',
    progress: 70,
    current_step: 'Running AI analysis',
    elapsed_ms: 12000
  }
}

POST /v1/bulk-analyze
Body: {
  usernames: ['nike', 'adidas', 'puma'],
  business_id: 'biz_123',
  analysis_type: 'light'
}
Response: {
  success: true,
  data: {
    batch_id: 'batch_xyz',
    total: 3,
    estimated_time: '60-90s'
  }
}
```

### **Success Criteria:**
- ✅ LIGHT analysis completes in <11s (cold)
- ✅ DEEP analysis completes in <25s (cold)
- ✅ XRAY analysis completes in <25s (cold)
- ✅ Cache hit rate >30% after 1 week
- ✅ Credit deduction atomic (no race conditions)
- ✅ Duplicate analysis check prevents double charge
- ✅ Lead upsert works correctly
- ✅ Bulk analysis processes batches correctly

---

## Phase 8: Async Orchestration (Weeks 7-8)

### **Build Order:**

```
core/workflows/
├── deep-analysis.workflow.ts
├── light-analysis.workflow.ts
├── xray-analysis.workflow.ts
├── bulk-analysis.workflow.ts
└── steps/
    ├── check-duplicate.step.ts     # Prevent duplicate analyses
    ├── check-cache.step.ts         # R2 cache lookup
    ├── scrape-profile.step.ts      # Apify with retries
    ├── run-ai-analysis.step.ts     # Parallel AI calls
    ├── cache-profile.step.ts       # Store in R2
    ├── upsert-lead.step.ts         # Update/insert lead
    └── save-results.step.ts        # Write to database

core/queues/
├── stripe-webhook.consumer.ts      # Process Stripe events
└── analysis.consumer.ts            # Trigger workflows

core/durable-objects/
└── analysis-progress.do.ts         # Real-time progress tracking
```

### **Workflow Pattern:**

```typescript
export class DeepAnalysisWorkflow extends WorkflowEntrypoint {
  async run(event, step) {
    const { run_id, username, account_id, business_id } = event.payload;
    
    // Get Durable Object for progress tracking
    const progressTracker = getProgressTracker(run_id);
    
    // Step 1: Check duplicate
    await updateProgress('queued', 5, 'Checking for duplicates');
    const duplicate = await step.do('check_duplicate', ...);
    if (duplicate) throw new Error('Already in progress');
    
    // Step 2: Check cache
    await updateProgress('scraping', 10, 'Checking cache');
    const cached = await step.do('check_cache', ...);
    
    // Step 3: Scrape (if cache miss)
    let profile;
    if (cached) {
      profile = cached.profile;
    } else {
      await updateProgress('scraping', 25, 'Scraping profile');
      profile = await step.do('scrape_profile', ...); // Retries on failure
      await step.do('cache_profile', ...);
    }
    
    // Step 4: Fetch business context
    await updateProgress('analyzing', 40, 'Loading business context');
    const business = await step.do('fetch_business', ...);
    
    // Step 5: AI analysis (parallel)
    await updateProgress('analyzing', 50, 'Running AI (3 parallel calls)');
    const [core, outreach, personality] = await step.do('ai_analysis', async () => {
      return await Promise.all([
        aiService.executeCoreStrategy(profile, business),
        aiService.generateOutreach(profile, business),
        aiService.analyzePersonality(profile)
      ]);
    });
    
    // Step 6: Upsert lead
    await updateProgress('saving', 80, 'Saving lead');
    const leadResult = await step.do('upsert_lead', ...);
    
    // Step 7: Save analysis
    await updateProgress('saving', 90, 'Saving results');
    await step.do('save_analysis', ...);
    
    // Complete
    await updateProgress('complete', 100, 'Analysis complete');
  }
}
```

### **Durable Object (Progress Tracker):**

```typescript
export class AnalysisProgressTracker extends DurableObject {
  async fetch(request) {
    const url = new URL(request.url);
    
    // GET /progress - Client polls
    if (url.pathname === '/progress') {
      const state = await this.ctx.storage.get('state');
      return Response.json(state);
    }
    
    // POST /update - Workflow updates
    if (url.pathname === '/update') {
      const update = await request.json();
      await this.ctx.storage.put('state', update);
      
      // Set cleanup alarm if complete
      if (['complete', 'failed'].includes(update.status)) {
        await this.ctx.storage.setAlarm(Date.now() + 3600000);
      }
      
      return Response.json({ ok: true });
    }
    
    // POST /cancel - Client cancels
    if (url.pathname === '/cancel') {
      const state = await this.ctx.storage.get('state');
      state.should_cancel = true;
      await this.ctx.storage.put('state', state);
      return Response.json({ ok: true });
    }
  }
  
  async alarm() {
    // Cleanup after 1 hour
    await this.ctx.storage.deleteAll();
  }
}
```

### **Queue Consumers:**

**Stripe Webhook Consumer:**
- Check idempotency (webhook_events table)
- Process event based on type:
  - `invoice.paid` → Record payment
  - `invoice.payment_failed` → Mark subscription past_due
  - `customer.subscription.created` → Confirm subscription
- Send to appropriate handler

**Analysis Consumer:**
- Receive analysis request from queue
- Trigger appropriate workflow (LIGHT/DEEP/XRAY)
- Handle workflow errors (refund credits)

### **Success Criteria:**
- ✅ Workflows complete without timeout (no 30s limit)
- ✅ Durable Objects track progress in real-time
- ✅ Client can poll progress every 2s
- ✅ Client can cancel in-progress analysis
- ✅ Stripe webhooks idempotent (no duplicate processing)
- ✅ Infrastructure failures auto-retry
- ✅ Business logic failures don't retry

---

## Phase 9: Observability & Monitoring (Week 9)

### **Build Order:**

```
infrastructure/monitoring/
├── analytics-engine.client.ts      # Cloudflare Analytics Engine
├── sentry.client.ts                # Sentry error tracking
├── error-classifier.ts             # Classify 4xx vs 5xx
└── retry-strategies.ts             # Exponential backoff configs

shared/middleware/
└── analytics.middleware.ts         # Track every request
```

### **Analytics Engine Integration:**

```typescript
// Track every analysis request
env.ANALYTICS_ENGINE.writeDataPoint({
  blobs: [run_id, username, analysisType, userId, accountId],
  doubles: [cost, duration_ms, score, cache_hit ? 1 : 0],
  indexes: [accountId]
});

// Track every API endpoint
env.ANALYTICS_ENGINE.writeDataPoint({
  blobs: [endpoint, method, status, userId],
  doubles: [duration_ms, response_size],
  indexes: [status]
});
```

### **Key Metrics to Track:**

**Performance:**
- P50, P95, P99 latency per endpoint
- Analysis duration by type (LIGHT/DEEP/XRAY)
- Cache hit rate over time
- Apify scraping duration
- AI call duration per model

**Cost:**
- Total cost per analysis (Apify + AI)
- Cost breakdown by provider (OpenAI, Anthropic, Apify)
- Cost per user/account
- Margin per analysis type

**Usage:**
- Analyses per day/week/month
- Most analyzed profiles (top 100)
- Busiest accounts (top 20)
- Analysis type distribution (LIGHT vs DEEP vs XRAY)

**Reliability:**
- Success rate per analysis type
- Failure reasons (out of credits, duplicate, timeout, scrape failed, AI error)
- Retry counts per step
- Refund frequency

### **Queries to Build:**

```typescript
// Which accounts drive 80% of AI costs?
SELECT account_id, SUM(cost) as total_cost
FROM analytics
WHERE blob1 = 'analysis'
GROUP BY account_id
ORDER BY total_cost DESC
LIMIT 20;

// P95 latency per endpoint
SELECT blob1 as endpoint, PERCENTILE_CONT(0.95) as p95_ms
FROM analytics
WHERE blob1 LIKE '/v1/%'
GROUP BY endpoint;

// Cache hit rate over time
SELECT DATE(timestamp) as date,
  SUM(CASE WHEN double4 = 1 THEN 1 ELSE 0 END) / COUNT(*) as hit_rate
FROM analytics
WHERE blob1 = 'analysis'
GROUP BY DATE(timestamp);
```

### **Success Criteria:**
- ✅ All requests logged to Analytics Engine
- ✅ Cost tracking accurate within 1%
- ✅ Can query P95 latency per endpoint
- ✅ Can identify top cost drivers
- ✅ Errors logged to Sentry with context

---

## Phase 10: Cron Jobs (Week 10)

### **Build Order:**

```
core/cron/
├── monthly-renewal.job.ts          # 1st of month, 3 AM UTC
├── daily-cleanup.job.ts            # Daily, 2 AM UTC
├── failed-analysis-cleanup.job.ts  # Hourly
└── subscription-sync.job.ts        # Every 6 hours (optional)
```

### **wrangler.toml Configuration:**

```toml
[triggers]
crons = [
  "0 3 1 * *",   # Monthly renewal
  "0 2 * * *",   # Daily cleanup
  "0 * * * *"    # Hourly failed analysis cleanup
]
```

### **Monthly Credit Renewal:**

```typescript
export async function monthlyRenewalJob(env) {
  // 1. Get renewable subscriptions
  const { data: subs } = await supabaseAdmin
    .rpc('get_renewable_subscriptions');
  
  let succeeded = 0, failed = 0;
  
  for (const sub of subs) {
    // 2. Idempotency check (23-hour window)
    const recentRenewal = await checkRecentRenewal(sub.account_id);
    if (recentRenewal) {
      succeeded++;
      continue;
    }
    
    try {
      // 3. Grant credits
      await supabaseAdmin.rpc('deduct_credits', {
        p_account_id: sub.account_id,
        p_amount: sub.credits_per_period, // Positive = grant
        p_operation_type: 'subscription_renewal',
        p_description: `Monthly ${sub.plan_name} renewal`
      });
      
      // 4. Advance subscription period
      const newEnd = new Date(sub.current_period_end);
      newEnd.setMonth(newEnd.getMonth() + 1);
      
      await supabaseAdmin
        .from('subscriptions')
        .update({
          current_period_start: sub.current_period_end,
          current_period_end: newEnd.toISOString()
        })
        .eq('id', sub.id);
      
      succeeded++;
    } catch (error) {
      console.error(`Renewal failed for ${sub.id}:`, error);
      failed++;
    }
  }
  
  console.log(`Renewed ${succeeded}/${subs.length}, ${failed} failed`);
}
```

### **Daily Cleanup:**

```typescript
export async function dailyCleanupJob(env) {
  // 1. Delete old AI logs (>90 days)
  await supabaseAdmin
    .from('ai_usage_logs')
    .delete()
    .lt('created_at', ninetyDaysAgo());
  
  // 2. Hard-delete soft-deleted records (>30 days)
  const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
  
  await supabaseAdmin
    .from('analyses')
    .delete()
    .lt('deleted_at', thirtyDaysAgo);
  
  await supabaseAdmin
    .from('leads')
    .delete()
    .lt('deleted_at', thirtyDaysAgo);
  
  // ... other tables
}
```

### **Failed Analysis Cleanup:**

```typescript
export async function failedAnalysisCleanupJob(env) {
  // Find stuck analyses (>5 minutes in pending/processing)
  const { data: stuck } = await supabaseAdmin
    .from('analyses')
    .select('id, account_id, credits_used')
    .in('status', ['pending', 'processing'])
    .lt('created_at', fiveMinutesAgo());
  
  for (const analysis of stuck) {
    // Mark as failed
    await supabaseAdmin
      .from('analyses')
      .update({
        status: 'failed',
        error_message: 'Analysis timed out after 5 minutes'
      })
      .eq('id', analysis.id);
    
    // Refund credits
    await supabaseAdmin.rpc('deduct_credits', {
      p_account_id: analysis.account_id,
      p_amount: -analysis.credits_used, // Negative = refund
      p_operation_type: 'timeout_refund',
      p_description: 'Auto-refund for timed out analysis'
    });
  }
}
```

### **Success Criteria:**
- ✅ Monthly renewal grants credits correctly
- ✅ Idempotency prevents double renewal
- ✅ Daily cleanup deletes old data
- ✅ Failed analyses refunded within 5 minutes
- ✅ Cron jobs logged to Analytics Engine

---

## Phase 11: Testing (Week 10-11)

### **Build Order:**

```
tests/
├── unit/
│   ├── features/
│   │   ├── credits/domain/credit-amount.vo.test.ts
│   │   ├── auth/application/verify-token.usecase.test.ts
│   │   ├── analysis/domain/lead-scorer.service.test.ts
│   │   └── business/domain/icp-matcher.service.test.ts
│   └── infrastructure/
│       ├── database/repositories/credits.repository.test.ts
│       └── cache/r2-cache.service.test.ts
├── integration/
│   ├── credits/balance-endpoint.test.ts
│   ├── analysis/analyze-endpoint.test.ts
│   └── business/crud-endpoints.test.ts
└── e2e/
    ├── full-analysis-workflow.test.ts
    └── bulk-analysis-workflow.test.ts
```

### **Testing Strategy:**

**Unit Tests (Domain Layer - 100% coverage):**
- Value objects validation
- Entity business logic
- Domain services (scoring, matching)
- Pure functions (no I/O)

**Unit Tests (Infrastructure - 70% coverage):**
- Repository methods (mock Supabase)
- Cache service (mock R2)
- AI adapters (mock OpenAI/Claude)
- Cost tracking (mock responses)

**Integration Tests (Key Flows - 90% coverage):**
- Credit balance endpoint (with test DB)
- Analysis creation endpoint (with test DB)
- Workflow steps (with mocked external APIs)

**E2E Tests (Critical Paths):**
- Full LIGHT analysis (scrape → AI → save)
- Full DEEP analysis (parallel AI calls)
- Bulk analysis (10 profiles)
- Credit deduction + refund flow

### **Test Configuration:**

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    pool: '@cloudflare/vitest-pool-workers',
    poolOptions: {
      workers: {
        miniflare: {
          bindings: {
            // Mock bindings for tests
          }
        }
      }
    }
  }
});
```

### **Example Test:**

```typescript
// tests/unit/features/credits/domain/credit-amount.vo.test.ts
import { describe, it, expect } from 'vitest';
import { CreditAmount } from '@/features/credits/domain/credit-amount.vo';

describe('CreditAmount Value Object', () => {
  it('should create valid positive amount', () => {
    const amount = new CreditAmount(10);
    expect(amount.value).toBe(10);
    expect(amount.isPositive()).toBe(true);
  });
  
  it('should create valid negative amount', () => {
    const amount = new CreditAmount(-5);
    expect(amount.value).toBe(-5);
    expect(amount.isNegative()).toBe(true);
  });
  
  it('should reject zero amount', () => {
    expect(() => new CreditAmount(0)).toThrow('Amount cannot be zero');
  });
  
  it('should reject decimal amount', () => {
    expect(() => new CreditAmount(1.5)).toThrow('Amount must be integer');
  });
});
```

### **Success Criteria:**
- ✅ Domain layer: 100% coverage
- ✅ Application layer: 90% coverage
- ✅ Infrastructure layer: 70% coverage
- ✅ All critical paths have E2E tests
- ✅ Tests run in <30 seconds
- ✅ CI/CD blocks deployment on test failure

---

## ✅ IMPLEMENTATION CHECKLIST

### **Phase 0: Foundation ✅**
- [ ] Repository initialized with proper structure
- [ ] All dependencies installed
- [ ] `wrangler.toml`, `tsconfig.json`, `package.json` configured
- [ ] Cloudflare bindings validated (KV, R2, Workflows, Queues, DO, Analytics)
- [ ] AWS Secrets Manager working with caching
- [ ] Supabase RLS performance fix applied (auth.uid() wrapped)
- [ ] All RPC functions have SECURITY DEFINER verified
- [ ] Health check endpoint returns 200 with binding status

### **Phase 1: Core Infrastructure ✅**
- [ ] AWS Secrets caching implemented
- [ ] Dual Supabase client factory (anon + service role)
- [ ] All repositories implemented (credits, business, leads, analysis)
- [ ] R2 cache service with TTL logic
- [ ] AI Gateway integration (OpenAI + Claude)
- [ ] Apify adapter with retry logic
- [ ] Cost tracker captures all expenses
- [ ] Performance tracker measures step durations

### **Phase 2: Shared Utilities ✅**
- [ ] Auth middleware validates JWT
- [ ] Rate limiting middleware (KV-based)
- [ ] Error middleware catches and formats errors
- [ ] Analytics middleware tracks all requests
- [ ] Logger utility outputs structured JSON
- [ ] Standard response format applied everywhere
- [ ] ID generator utility for run_id, lead_id

### **Phase 3: Domain Models ✅**
- [ ] Analysis config centralized (LIGHT/DEEP/XRAY)
- [ ] Analysis entity with business logic
- [ ] Lead entity with validation
- [ ] Value objects (score, handle, type)
- [ ] Domain services (scorer, duplicate checker)

### **Phase 4: Credits Feature ✅**
- [ ] GET /credits/balance endpoint
- [ ] GET /credits/transactions endpoint
- [ ] Credit balance returns correct value
- [ ] Transaction history paginated
- [ ] RLS ensures users only see own data

### **Phase 5: Auth Feature ✅**
- [ ] JWT validation service
- [ ] Token verification use case
- [ ] GET /auth/verify endpoint
- [ ] Middleware reuses auth logic

### **Phase 6: Business Profiles ✅**
- [ ] Business profile entity with ICP
- [ ] CRUD endpoints (create, read, update, delete)
- [ ] Soft delete preserves relationships
- [ ] ICP matcher service

### **Phase 7: Analysis Feature ✅**
- [ ] 12-step analysis flow implemented
- [ ] POST /v1/analyze endpoint
- [ ] POST /v1/bulk-analyze endpoint
- [ ] GET /runs/:run_id endpoint
- [ ] GET /runs/:run_id/progress endpoint
- [ ] AI analysis service (LIGHT/DEEP/XRAY)
- [ ] Prompt builder with caching
- [ ] Duplicate check prevents double charge
- [ ] Lead upsert works correctly
- [ ] Cache hit rate tracking

### **Phase 8: Async Orchestration ✅**
- [ ] LIGHT analysis workflow
- [ ] DEEP analysis workflow
- [ ] XRAY analysis workflow
- [ ] Bulk analysis workflow
- [ ] Durable Object progress tracker
- [ ] Stripe webhook queue consumer
- [ ] Analysis queue consumer
- [ ] Workflow steps modularized
- [ ] Cancel functionality works

### **Phase 9: Observability ✅**
- [ ] Analytics Engine tracking all requests
- [ ] Cost metrics accurate
- [ ] Performance metrics captured
- [ ] Sentry error tracking configured
- [ ] Can query P95 latency
- [ ] Can identify top cost drivers

### **Phase 10: Cron Jobs ✅**
- [ ] Monthly renewal job (1st, 3 AM UTC)
- [ ] Daily cleanup job (2 AM UTC)
- [ ] Failed analysis cleanup (hourly)
- [ ] Idempotency checks working
- [ ] Jobs logged to Analytics Engine

### **Phase 11: Testing ✅**
- [ ] Domain unit tests (100% coverage)
- [ ] Application unit tests (90% coverage)
- [ ] Infrastructure unit tests (70% coverage)
- [ ] Integration tests for key flows
- [ ] E2E tests for critical paths
- [ ] All tests passing
- [ ] Test suite runs in <30s

---

## 🚀 DEPLOYMENT CHECKLIST

### **Pre-Deployment:**
- [ ] All phases complete
- [ ] All tests passing
- [ ] Staging deployment successful
- [ ] Load testing completed
- [ ] Security audit passed
- [ ] Database migrations applied
- [ ] AWS Secrets populated
- [ ] Cloudflare bindings configured

### **Deployment:**
- [ ] Deploy to production: `npm run deploy`
- [ ] Verify health check: `curl https://api.oslira.com/health`
- [ ] Test analysis endpoint with test account
- [ ] Verify cron jobs scheduled
- [ ] Check Analytics Engine receiving data
- [ ] Monitor errors in Sentry

### **Post-Deployment:**
- [ ] Monitor performance for 24 hours
- [ ] Verify cache hit rate increasing
- [ ] Check cost tracking accurate
- [ ] Confirm no duplicate analyses
- [ ] Validate credit deductions atomic
- [ ] Test cancel functionality

---

**Total Implementation Time: 10-11 Weeks**

**Phases 0-3:** Weeks 1-3 (Foundation + Infrastructure + Shared + Domain)  
**Phases 4-7:** Weeks 4-6 (Features: Credits, Auth, Business, Analysis)  
**Phases 8-10:** Weeks 7-10 (Async + Observability + Cron)  
**Phase 11:** Week 10-11 (Testing)

---

## 🎯 SUCCESS METRICS

**Performance:**
| Metric | Current | Target |
|--------|---------|--------|
| LIGHT cold | 10-13s | 8-11s |
| LIGHT cached | N/A | 2-3s |
| DEEP cold | 30s | 18-23s |
| XRAY cold | 35s | 16-18s |
| Bulk 100 | 16min | 60-120s |
| Credit check | 500ms | <50ms |

**Cost:**
| Item | Current | Target | Savings |
|------|---------|--------|---------|
| AI | $300/mo | $210/mo | $90/mo |
| Worker CPU | $100/mo | $2/mo | $98/mo |
| **Total** | **$400/mo** | **$212/mo** | **$188/mo** |

**Reliability:**
| Metric | Current | Target |
|--------|---------|--------|
| Workflow success | N/A | >99% |
| Webhook loss | Possible | 0% (Queue retries) |
| Analysis failures | Lost forever | Auto-retry + refund |

---

## 🏗️ ARCHITECTURE PRINCIPLES

1. **Feature-first** - Each feature owns domain/application/controllers
2. **Ports & Adapters** - Domain defines interfaces, infrastructure implements
3. **Dependency Rule** - Outer depends on inner, never reverse
4. **Single Responsibility** - Each module does ONE thing
5. **Fail-safe Defaults** - Default deny, explicit grant
6. **Observable** - Every request tracked, every error classified
7. **Account-Based** - Credits belong to accounts, not users
8. **Idempotent** - Webhooks, crons can run multiple times safely

---

## ✅ IMPLEMENTATION CHECKLIST

### **Infrastructure:**
- [ ] Validate Cloudflare bindings (KV, R2, Workflows, Queues, DO)
- [ ] Apply RLS fix to Supabase (wrap auth.uid())
- [ ] Add cron triggers to wrangler.toml
- [ ] Configure Analytics Engine binding
- [ ] Set up Sentry project

### **Database:**
- [ ] Verify `get_renewable_subscriptions()` exists
- [ ] Verify `deduct_credits()` exists
- [ ] Add indexes (analyses.status, ai_usage_logs.created_at)
- [ ] Verify SECURITY DEFINER on all RPC functions
- [ ] Test cascading deletes

### **Features:**
- [ ] Build credits feature (validates pattern)
- [ ] Build auth feature
- [ ] Build business profiles feature
- [ ] Build analysis feature (core product)

### **Infrastructure Adapters:**
- [ ] Dual Supabase client factory
- [ ] Lead upsert repository method
- [ ] R2 cache service
- [ ] AI Gateway integration
- [ ] Cost tracker
- [ ] Performance tracker

### **Async Infrastructure:**
- [ ] Deep analysis workflow
- [ ] Light analysis workflow
- [ ] XRAY analysis workflow
- [ ] Bulk analysis workflow
- [ ] Stripe webhook queue consumer
- [ ] Analysis progress Durable Object

### **Cron Jobs:**
- [ ] Monthly credit renewal
- [ ] Daily cleanup
- [ ] Failed analysis cleanup
- [ ] Manual trigger admin endpoints

### **Critical Fixes:**
- [ ] Webhook idempotency check
- [ ] Change user_id to account_id everywhere
- [ ] Duplicate analysis check before deduction

### **Testing:**
- [ ] Unit tests (domain layer 100%)
- [ ] Integration tests (key flows)
- [ ] Test in staging before production
- [ ] Verify idempotency (run crons twice)

---

## 🚀 DEPLOYMENT APPROACH

**Single-Pass Implementation:**
1. Build all phases in new repository
2. Copy working code from old worker
3. Split into feature-first structure
4. Test thoroughly in staging
5. Deploy all at once to production
6. Monitor via Analytics Engine + Sentry

**Not doing incremental rollout** - Team size supports all-at-once deployment.

---

## 📊 FINAL RATING

**Current Worker: 3/10**
- Blocking synchronous operations
- Security holes (RLS bypassed)
- Zero observability
- Poor architecture
- Race conditions
- Wrong multi-tenancy model

**Target Worker: 97/100**
- Async orchestration (no timeouts)
- Defense-in-depth security
- Full observability
- Clean feature-first architecture
- Comprehensive cost tracking
- Production-ready with all optimizations

**Why not 100/100?** Final 3 points earned through production battle-testing and real-world edge case handling.

---

**END OF ARCHITECTURE SPECIFICATION**

---

# 💻 CODE IMPLEMENTATION REFERENCE

## SECTION 1: AWS SECRETS & CONFIGURATION

### **Required AWS Secrets:**
```
SUPABASE_URL
SUPABASE_ANON_KEY
SUPABASE_SERVICE_ROLE_KEY
APIFY_API_TOKEN
OPENAI_API_KEY
ANTHROPIC_API_KEY
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET
SENTRY_DSN
```

### **wrangler.toml Complete Configuration:**
```toml
name = "oslira-workers"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[build.upload]
format = "modules"

[vars]
AWS_ACCESS_KEY_ID = "AKIA..."
AWS_SECRET_ACCESS_KEY = "..."
AWS_REGION = "us-east-1"
APP_ENV = "production"
CLOUDFLARE_ACCOUNT_ID = "..."
AI_GATEWAY_NAME = "oslira-gateway"

[[kv_namespaces]]
binding = "OSLIRA_KV"
id = "..."

[[r2_buckets]]
binding = "R2_CACHE_BUCKET"
bucket_name = "oslira-cache"

[[workflows]]
binding = "ANALYSIS_WORKFLOW"
name = "deep-analysis-workflow"
class_name = "DeepAnalysisWorkflow"

[[queues.producers]]
binding = "ANALYSIS_QUEUE"
queue = "analysis-queue"

[[queues.consumers]]
queue = "analysis-queue"
max_batch_size = 10
max_batch_timeout = 30

[[queues.producers]]
binding = "STRIPE_WEBHOOKS_QUEUE"
queue = "stripe-webhooks"

[[queues.consumers]]
queue = "stripe-webhooks"
max_batch_size = 10
max_batch_timeout = 10

[durable_objects]
bindings = [
  { name = "ANALYSIS_PROGRESS", class_name = "AnalysisProgressTracker" }
]

[[migrations]]
tag = "v1"
new_classes = ["AnalysisProgressTracker"]

[[analytics_engine_datasets]]
binding = "ANALYTICS_ENGINE"

[triggers]
crons = [
  "0 3 1 * *",
  "0 2 * * *",
  "0 * * * *"
]
```

### **package.json:**
```json
{
  "name": "oslira-workers",
  "version": "6.0.0",
  "scripts": {
    "dev": "wrangler dev",
    "deploy": "wrangler deploy",
    "test": "vitest",
    "test:unit": "vitest run tests/unit",
    "test:integration": "vitest run tests/integration",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "hono": "^4.0.0",
    "@supabase/supabase-js": "^2.39.0",
    "openai": "^4.28.0",
    "@anthropic-ai/sdk": "^0.17.0",
    "stripe": "^14.0.0",
    "zod": "^3.22.0",
    "@aws-sdk/client-secrets-manager": "^3.490.0"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "^4.20240117.0",
    "@cloudflare/vitest-pool-workers": "^0.1.0",
    "vitest": "^1.2.0",
    "typescript": "^5.3.0",
    "wrangler": "^3.25.0"
  }
}
```

### **tsconfig.json:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "lib": ["ES2022"],
    "moduleResolution": "bundler",
    "types": ["@cloudflare/workers-types"],
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "strict": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@features/*": ["src/features/*"],
      "@core/*": ["src/core/*"],
      "@infrastructure/*": ["src/infrastructure/*"],
      "@shared/*": ["src/shared/*"]
    }
  },
  "include": ["src/**/*", "tests/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## SECTION 2: CORE INFRASTRUCTURE

### **Dual Supabase Client Factory:**
```typescript
// infrastructure/database/supabase.client.ts
import { createClient, SupabaseClient } from '@supabase/supabase-js';
import { getApiKey } from '@/infrastructure/config/secrets';

export class SupabaseClientFactory {
  private static userClientCache: Map<string, SupabaseClient> = new Map();
  private static adminClient: SupabaseClient | null = null;

  static async createUserClient(env: Env): Promise<SupabaseClient> {
    const url = await getApiKey('SUPABASE_URL', env, env.APP_ENV);
    const anonKey = await getApiKey('SUPABASE_ANON_KEY', env, env.APP_ENV);
    
    const cacheKey = `${url}:${anonKey}`;
    if (!this.userClientCache.has(cacheKey)) {
      this.userClientCache.set(cacheKey, createClient(url, anonKey));
    }
    
    return this.userClientCache.get(cacheKey)!;
  }

  static async createAdminClient(env: Env): Promise<SupabaseClient> {
    if (this.adminClient) return this.adminClient;
    
    const url = await getApiKey('SUPABASE_URL', env, env.APP_ENV);
    const serviceRoleKey = await getApiKey('SUPABASE_SERVICE_ROLE_KEY', env, env.APP_ENV);
    
    this.adminClient = createClient(url, serviceRoleKey, {
      auth: { persistSession: false }
    });
    
    return this.adminClient;
  }
}
```

### **AWS Secrets Manager with Caching:**
```typescript
// infrastructure/config/secrets.ts
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const secretsCache = new Map<string, { value: string; cachedAt: number }>();
const CACHE_TTL = 5 * 60 * 1000; // 5 minutes

export async function getApiKey(
  secretName: string,
  env: Env,
  appEnv: string
): Promise<string> {
  const cacheKey = `${appEnv}:${secretName}`;
  const cached = secretsCache.get(cacheKey);
  
  if (cached && (Date.now() - cached.cachedAt) < CACHE_TTL) {
    return cached.value;
  }
  
  const client = new SecretsManagerClient({
    region: env.AWS_REGION,
    credentials: {
      accessKeyId: env.AWS_ACCESS_KEY_ID,
      secretAccessKey: env.AWS_SECRET_ACCESS_KEY
    }
  });
  
  const command = new GetSecretValueCommand({
    SecretId: `${appEnv}/${secretName}`
  });
  
  const response = await client.send(command);
  const value = response.SecretString!;
  
  secretsCache.set(cacheKey, { value, cachedAt: Date.now() });
  
  return value;
}
```

### **R2 Cache Service:**
```typescript
// infrastructure/cache/r2-cache.service.ts
export class R2CacheService {
  constructor(private bucket: R2Bucket) {}

  async get(username: string, analysisType: 'light' | 'deep' | 'xray'): Promise<CachedProfile | null> {
    const key = `instagram:${username}:v1`;
    const cached = await this.bucket.get(key);
    
    if (!cached) return null;
    
    const data = await cached.json() as CachedProfile;
    const age = Date.now() - new Date(data.cached_at).getTime();
    
    const ttls = {
      light: 24 * 60 * 60 * 1000,
      deep: 12 * 60 * 60 * 1000,
      xray: 6 * 60 * 60 * 1000
    };
    
    if (age > ttls[analysisType]) {
      return null;
    }
    
    return data;
  }

  async set(username: string, profileData: ProfileData): Promise<void> {
    const key = `instagram:${username}:v1`;
    const cacheData = {
      profile: profileData,
      cached_at: new Date().toISOString(),
      username: profileData.username,
      followers: profileData.followersCount
    };
    
    await this.bucket.put(key, JSON.stringify(cacheData));
  }

  async invalidate(username: string): Promise<void> {
    const key = `instagram:${username}:v1`;
    await this.bucket.delete(key);
  }
}

interface CachedProfile {
  profile: ProfileData;
  cached_at: string;
  username: string;
  followers: number;
}
```

### **AI Gateway Client:**
```typescript
// infrastructure/ai/ai-gateway.client.ts
import { OpenAI } from 'openai';

export class AIGatewayClient {
  private openai: OpenAI;

  constructor(env: Env) {
    const baseURL = `https://gateway.ai.cloudflare.com/v1/${env.CLOUDFLARE_ACCOUNT_ID}/${env.AI_GATEWAY_NAME}/openai`;
    
    this.openai = new OpenAI({
      apiKey: env.OPENAI_API_KEY, // From AWS Secrets
      baseURL
    });
  }

  async createChatCompletion(params: OpenAI.Chat.CompletionCreateParams) {
    return await this.openai.chat.completions.create(params);
  }
}
```

---

## SECTION 3: REPOSITORY LAYER

### **Lead Upsert Repository:**
```typescript
// infrastructure/database/repositories/leads.repository.ts
export class LeadsRepository {
  constructor(private supabase: SupabaseClient) {}

  async upsertLead(data: UpsertLeadData): Promise<UpsertLeadResult> {
    const { data: result, error } = await this.supabase.rpc('upsert_lead', {
      p_username: data.username,
      p_account_id: data.accountId,
      p_business_profile_id: data.businessProfileId,
      p_follower_count: data.followerCount,
      p_bio: data.bio,
      p_profile_pic_url: data.profilePicUrl,
      p_instagram_url: data.instagramUrl,
      p_is_verified: data.isVerified
    });

    if (error) throw error;

    return {
      leadId: result.id,
      isNew: result.is_new
    };
  }
}

interface UpsertLeadData {
  username: string;
  accountId: string;
  businessProfileId: string;
  followerCount: number;
  bio: string;
  profilePicUrl: string;
  instagramUrl: string;
  isVerified: boolean;
}

interface UpsertLeadResult {
  leadId: string;
  isNew: boolean;
}
```

**Supabase SQL Function:**
```sql
```sql
CREATE OR REPLACE FUNCTION upsert_lead(
  p_username TEXT,
  p_account_id UUID,
  p_business_profile_id UUID,
  p_follower_count INTEGER,
  p_bio TEXT,
  p_profile_pic_url TEXT,
  p_instagram_url TEXT,
  p_is_verified BOOLEAN
) RETURNS TABLE(id UUID, is_new BOOLEAN) AS $$
BEGIN
  RETURN QUERY
  INSERT INTO leads (
    username, account_id, business_profile_id,
    follower_count, bio, profile_pic_url, instagram_url,
    is_verified, first_analyzed_at, last_analyzed_at
  )
  VALUES (
    p_username, p_account_id, p_business_profile_id,
    p_follower_count, p_bio, p_profile_pic_url, p_instagram_url,
    p_is_verified, NOW(), NOW()
  )
  ON CONFLICT (username, account_id, business_profile_id)
  WHERE deleted_at IS NULL
  DO UPDATE SET
    follower_count = EXCLUDED.follower_count,
    bio = EXCLUDED.bio,
    profile_pic_url = EXCLUDED.profile_pic_url,
    instagram_url = COALESCE(leads.instagram_url, EXCLUDED.instagram_url),
    is_verified = EXCLUDED.is_verified,
    last_analyzed_at = NOW()
  RETURNING leads.id, (xmax = 0) AS is_new;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## SECTION 4: ANALYSIS CONFIGURATION

### **Centralized Analysis Config:**
```typescript
// features/analysis/domain/config/analysis.config.ts
export const ANALYSIS_CONFIG = {
  LIGHT: {
    internal_name: 'light',
    display_name: 'Quick Analysis',
    db_value: 'light',
    credit_cost: 1,
    price_usd: 0.97,
    
    performance: {
      target_duration_ms: 6000,
      acceptable_max_ms: 11000,
      cache_ttl_hours: 24
    },
    
    ai: {
      model: 'gpt-4o-mini',
      reasoning_effort: 'low',
      max_tokens: 1000,
      temperature: 0,
      use_prompt_cache: true
    },
    
    scraping: {
      required: true,
      posts_limit: 12,
      use_cache: true
    },
    
    output: {
      includes: ['score', 'brief_summary', 'confidence'],
      excludes: ['deep_insights', 'outreach']
    }
  },
  
  DEEP: {
    internal_name: 'deep',
    display_name: 'Profile Analysis',
    db_value: 'deep',
    credit_cost: 5,
    price_usd: 4.85,
    
    performance: {
      target_duration_ms: 20000,
      acceptable_max_ms: 25000,
      cache_ttl_hours: 12
    },
    
    ai: {
      parallel_calls: 3,
      models: {
        core_strategy: 'gpt-5-mini',
        outreach: 'gpt-5-mini',
        personality: 'gpt-5-mini'
      },
      reasoning_efforts: {
        core_strategy: 'medium',
        outreach: 'low',
        personality: 'low'
      },
      use_prompt_cache: true
    },
    
    scraping: {
      required: true,
      posts_limit: 50,
      use_cache: true
    },
    
    output: {
      includes: ['score', 'deep_summary', 'outreach', 'personality', 'selling_points'],
      excludes: ['psychographic_deep_dive']
    }
  },
  
  XRAY: {
    internal_name: 'xray',
    display_name: 'X-Ray Analysis',
    db_value: 'xray',
    credit_cost: 6,
    price_usd: 5.82,
    
    performance: {
      target_duration_ms: 18000,
      acceptable_max_ms: 25000,
      cache_ttl_hours: 6
    },
    
    ai: {
      parallel_calls: 4,
      models: {
        psychographic: 'gpt-5',
        commercial: 'gpt-5-mini',
        outreach: 'gpt-5-mini',
        personality: 'gpt-5-mini'
      },
      reasoning_efforts: {
        psychographic: 'medium',
        commercial: 'medium',
        outreach: 'low',
        personality: 'low'
      },
      use_prompt_cache: true
    },
    
    scraping: {
      required: true,
      posts_limit: 50,
      use_cache: true
    },
    
    output: {
      includes: ['all'],
      excludes: []
    }
  }
} as const;

export type AnalysisType = 'light' | 'deep' | 'xray';

export function getAnalysisConfig(type: AnalysisType) {
  return ANALYSIS_CONFIG[type.toUpperCase() as keyof typeof ANALYSIS_CONFIG];
}
```

### **AI Pricing Config:**
```typescript
// infrastructure/ai/pricing.config.ts
export const AI_PRICING = {
  'gpt-4o-mini': {
    per_1m_input: 0.15,
    per_1m_output: 0.60
  },
  'gpt-5-mini': {
    per_1m_input: 0.30,
    per_1m_output: 1.20
  },
  'gpt-5': {
    per_1m_input: 0.60,
    per_1m_output: 2.40
  },
  'claude-3-5-sonnet': {
    per_1m_input: 3.00,
    per_1m_output: 15.00
  }
} as const;
```

---

## SECTION 5: COST & PERFORMANCE TRACKING

### **Cost Tracker Service:**
```typescript
// infrastructure/monitoring/cost-tracker.service.ts
export class CostTracker {
  private costs = {
    apify: 0,
    ai_calls: [] as AICallCost[],
    total: 0
  };

  trackApifyCall(durationMs: number) {
    const computeUnits = durationMs / (1000 * 60 * 60);
    const cost = computeUnits * 0.25;
    this.costs.apify += cost;
    this.costs.total += cost;
  }

  trackAICall(model: string, tokensIn: number, tokensOut: number, durationMs: number) {
    const pricing = AI_PRICING[model as keyof typeof AI_PRICING];
    if (!pricing) throw new Error(`Unknown model: ${model}`);
    
    const cost = 
      (tokensIn / 1_000_000) * pricing.per_1m_input +
      (tokensOut / 1_000_000) * pricing.per_1m_output;
    
    this.costs.ai_calls.push({
      model,
      tokens_in: tokensIn,
      tokens_out: tokensOut,
      cost,
      duration_ms: durationMs
    });
    
    this.costs.total += cost;
  }

  getBreakdown() {
    const costByProvider = {
      apify: this.costs.apify,
      openai: this.costs.ai_calls
        .filter(c => c.model.includes('gpt'))
        .reduce((sum, c) => sum + c.cost, 0),
      claude: this.costs.ai_calls
        .filter(c => c.model.includes('claude'))
        .reduce((sum, c) => sum + c.cost, 0)
    };

    return {
      apify_cost: this.costs.apify,
      ai_costs: this.costs.ai_calls,
      total_cost: this.costs.total,
      cost_by_provider: costByProvider,
      margins: {
        light: 0.97 - this.costs.total,
        deep: 4.85 - this.costs.total,
        xray: 5.82 - this.costs.total
      }
    };
  }
}

interface AICallCost {
  model: string;
  tokens_in: number;
  tokens_out: number;
  cost: number;
  duration_ms: number;
}
```

### **Performance Tracker Service:**
```typescript
// infrastructure/monitoring/performance-tracker.service.ts
export class PerformanceTracker {
  private steps: PerformanceStep[] = [];

  startStep(name: string) {
    this.steps.push({
      name,
      start: Date.now(),
      end: null,
      duration_ms: null
    });
  }

  endStep(name: string) {
    const step = this.steps.find(s => s.name === name && s.end === null);
    if (step) {
      step.end = Date.now();
      step.duration_ms = step.end - step.start;
    }
  }

  getBreakdown() {
    const total = this.steps.reduce((sum, s) => sum + (s.duration_ms || 0), 0);
    const bottleneck = this.steps.reduce((max, s) =>
      (s.duration_ms || 0) > (max.duration_ms || 0) ? s : max
    );

    return {
      steps: this.steps.map(s => ({
        step: s.name,
        duration_ms: s.duration_ms,
        percentage: total > 0 ? ((s.duration_ms || 0) / total) * 100 : 0
      })),
      total_duration_ms: total,
      bottleneck: {
        step: bottleneck.name,
        duration_ms: bottleneck.duration_ms
      }
    };
  }
}

interface PerformanceStep {
  name: string;
  start: number;
  end: number | null;
  duration_ms: number | null;
}
```

---

## SECTION 6: DURABLE OBJECT IMPLEMENTATION

### **Analysis Progress Durable Object:**
```typescript
// core/durable-objects/analysis-progress.do.ts
import { DurableObject } from 'cloudflare:workers';

export class AnalysisProgressTracker extends DurableObject {
  private state: AnalysisProgressState;

  constructor(state: DurableObjectState, env: Env) {
    super(state, env);
    this.state = {
      run_id: '',
      status: 'queued',
      progress: 0,
      current_step: '',
      elapsed_ms: 0,
      should_cancel: false,
      created_at: Date.now()
    };
  }

  async fetch(request: Request) {
    const url = new URL(request.url);

    // GET /progress - Client polls this
    if (request.method === 'GET' && url.pathname === '/progress') {
      await this.loadState();
      return Response.json({
        run_id: this.state.run_id,
        status: this.state.status,
        progress: this.state.progress,
        current_step: this.state.current_step,
        elapsed_ms: Date.now() - this.state.created_at
      });
    }

    // POST /update - Workflow sends updates here
    if (request.method === 'POST' && url.pathname === '/update') {
      const update = await request.json() as Partial<AnalysisProgressState>;
      this.state = { ...this.state, ...update };
      await this.ctx.storage.put('state', this.state);

      // Set cleanup alarm if complete
      if (['complete', 'failed', 'cancelled'].includes(this.state.status)) {
        await this.ctx.storage.setAlarm(Date.now() + 3600000); // 1 hour
      }

      return Response.json({ ok: true });
    }

    // POST /cancel - Client cancels analysis
    if (request.method === 'POST' && url.pathname === '/cancel') {
      this.state.should_cancel = true;
      this.state.status = 'cancelled';
      await this.ctx.storage.put('state', this.state);
      return Response.json({ ok: true, message: 'Cancel signal sent' });
    }

    // GET /should-cancel - Workflow checks this between steps
    if (request.method === 'GET' && url.pathname === '/should-cancel') {
      await this.loadState();
      return Response.json({ should_cancel: this.state.should_cancel });
    }

    return new Response('Not Found', { status: 404 });
  }

  async alarm() {
    // Cleanup: Delete state 1 hour after completion
    await this.ctx.storage.deleteAll();
  }

  private async loadState() {
    const stored = await this.ctx.storage.get<AnalysisProgressState>('state');
    if (stored) {
      this.state = stored;
    }
  }
}

interface AnalysisProgressState {
  run_id: string;
  status: 'queued' | 'scraping' | 'analyzing' | 'saving' | 'complete' | 'failed' | 'cancelled';
  progress: number;
  current_step: string;
  elapsed_ms: number;
  should_cancel: boolean;
  created_at: number;
}
```

---

## SECTION 7: WORKFLOW IMPLEMENTATION

### **Deep Analysis Workflow:**
```typescript
// core/workflows/deep-analysis.workflow.ts
import { WorkflowEntrypoint, WorkflowEvent, WorkflowStep } from 'cloudflare:workers';

export class DeepAnalysisWorkflow extends WorkflowEntrypoint<Env, AnalysisParams> {
  async run(event: WorkflowEvent<AnalysisParams>, step: WorkflowStep) {
    const { run_id, username, business_id, account_id } = event.payload;

    // Get Durable Object stub
    const doId = this.env.ANALYSIS_PROGRESS.idFromName(run_id);
    const progressTracker = this.env.ANALYSIS_PROGRESS.get(doId);

    // Helper: Update progress
    const updateProgress = async (
      status: string,
      progress: number,
      current_step: string
    ) => {
      await progressTracker.fetch('https://fake-host/update', {
        method: 'POST',
        body: JSON.stringify({ status, progress, current_step })
      });
    };

    // Helper: Check if cancelled
    const shouldCancel = async (): Promise<boolean> => {
      const response = await progressTracker.fetch('https://fake-host/should-cancel');
      const data = await response.json() as { should_cancel: boolean };
      return data.should_cancel;
    };

    try {
      // Initialize
      await updateProgress('queued', 0, 'Starting analysis');

      // Step 1: Check for duplicate analysis
      if (await shouldCancel()) throw new Error('CANCELLED');
      await updateProgress('queued', 5, 'Checking for duplicate analysis');

      const duplicate = await step.do('check_duplicate', async () => {
        const supabase = await SupabaseClientFactory.createAdminClient(this.env);
        const { data } = await supabase
          .from('analyses')
          .select('run_id')
          .eq('username', username)
          .eq('account_id', account_id)
          .in('status', ['pending', 'processing'])
          .gte('created_at', new Date(Date.now() - 5 * 60 * 1000).toISOString())
          .limit(1)
          .single();
        return data;
      });

      if (duplicate) {
        throw new Error('Analysis already in progress for this profile');
      }

      // Step 2: Check cache
      if (await shouldCancel()) throw new Error('CANCELLED');
      await updateProgress('scraping', 10, 'Checking cache');

      const cached = await step.do('check_cache', async () => {
        const cacheService = new R2CacheService(this.env.R2_CACHE_BUCKET);
        return await cacheService.get(username, 'deep');
      });

      let profile: ProfileData;

      if (cached) {
        await updateProgress('analyzing', 30, 'Loaded from cache');
        profile = cached.profile;
      } else {
        // Step 3: Scrape profile
        if (await shouldCancel()) throw new Error('CANCELLED');
        await updateProgress('scraping', 25, 'Scraping Instagram profile');

        profile = await step.do('scrape_profile', async () => {
          const apifyKey = await getApiKey('APIFY_API_TOKEN', this.env, this.env.APP_ENV);
          const apifyAdapter = new ApifyAdapter(apifyKey);
          return await apifyAdapter.scrapeProfile(username, 50);
        });

        // Step 4: Cache profile
        await step.do('cache_profile', async () => {
          const cacheService = new R2CacheService(this.env.R2_CACHE_BUCKET);
          await cacheService.set(username, profile);
        });
      }

      // Step 5: Fetch business profile
      if (await shouldCancel()) throw new Error('CANCELLED');
      await updateProgress('analyzing', 40, 'Loading business context');

      const business = await step.do('fetch_business', async () => {
        const supabase = await SupabaseClientFactory.createUserClient(this.env);
        const { data, error } = await supabase
          .from('business_profiles')
          .select('*')
          .eq('id', business_id)
          .eq('account_id', account_id)
          .single();

        if (error || !data) throw new Error('Business profile not found');
        return data;
      });

      // Step 6: Run AI analysis (parallel)
      if (await shouldCancel()) throw new Error('CANCELLED');
      await updateProgress('analyzing', 50, 'Running AI analysis (3 parallel calls)');

      const [coreAnalysis, outreach, personality] = await step.do('ai_analysis', async () => {
        const aiGateway = new AIGatewayClient(this.env);
        const aiService = new AIAnalysisService(aiGateway);

        return await Promise.all([
          aiService.executeCoreStrategy(profile, business),     // 15s
          aiService.generateOutreachMessage(profile, business), // 4s
          aiService.analyzePersonality(profile)                 // 4s
        ]);
        // Total time: max(15, 4, 4) = 15s
      });

      // Step 7: Upsert lead
      if (await shouldCancel()) throw new Error('CANCELLED');
      await updateProgress('saving', 80, 'Saving lead data');

      const leadResult = await step.do('upsert_lead', async () => {
        const supabase = await SupabaseClientFactory.createAdminClient(this.env);
        const leadsRepo = new LeadsRepository(supabase);

        return await leadsRepo.upsertLead({
          username: profile.username,
          accountId: account_id,
          businessProfileId: business_id,
          followerCount: profile.followersCount,
          bio: profile.biography,
          profilePicUrl: profile.profilePicUrl,
          instagramUrl: profile.url,
          isVerified: profile.isVerified
        });
      });

      // Step 8: Save analysis results
      if (await shouldCancel()) throw new Error('CANCELLED');
      await updateProgress('saving', 90, 'Saving analysis results');

      await step.do('save_analysis', async () => {
        const supabase = await SupabaseClientFactory.createAdminClient(this.env);
        const analysisRepo = new AnalysisRepository(supabase);

        await analysisRepo.create({
          run_id,
          lead_id: leadResult.leadId,
          account_id,
          business_profile_id: business_id,
          analysis_type: 'deep',
          score: coreAnalysis.score,
          core_strategy: coreAnalysis,
          outreach_message: outreach,
          personality_analysis: personality,
          status: 'complete'
        });
      });

      // Complete
      await updateProgress('complete', 100, 'Analysis complete');

    } catch (error: any) {
      if (error.message === 'CANCELLED') {
        await updateProgress('cancelled', 0, 'Analysis cancelled by user');
      } else if (error.message.includes('duplicate')) {
        await updateProgress('failed', 0, error.message);
      } else {
        await updateProgress('failed', 0, `Error: ${error.message}`);
        
        // Refund credits on infrastructure failure
        await step.do('refund_credits', async () => {
          const supabase = await SupabaseClientFactory.createAdminClient(this.env);
          await supabase.rpc('deduct_credits', {
            p_account_id: account_id,
            p_amount: -5, // Negative = refund
            p_operation_type: 'analysis_failure_refund',
            p_description: 'Auto-refund for failed analysis'
          });
        });
      }
      throw error;
    }
  }
}

interface AnalysisParams {
  run_id: string;
  username: string;
  business_id: string;
  account_id: string;
}

interface ProfileData {
  username: string;
  followersCount: number;
  biography: string;
  profilePicUrl: string;
  url: string;
  isVerified: boolean;
}
```

---

## SECTION 8: WEBHOOK IDEMPOTENCY

### **Stripe Webhook Handler:**
```typescript
// features/billing/controllers/webhook.controller.ts
import Stripe from 'stripe';

export async function handleStripeWebhook(request: Request, env: Env): Promise<Response> {
  // 1. Verify signature
  const signature = request.headers.get('stripe-signature');
  if (!signature) {
    return new Response('Missing signature', { status: 400 });
  }

  const payload = await request.text();
  const webhookSecret = await getApiKey('STRIPE_WEBHOOK_SECRET', env, env.APP_ENV);

  let event: Stripe.Event;
  try {
    const stripeKey = await getApiKey('STRIPE_SECRET_KEY', env, env.APP_ENV);
    const stripe = new Stripe(stripeKey);
    event = stripe.webhooks.constructEvent(payload, signature, webhookSecret);
  } catch (err: any) {
    console.error('Webhook signature verification failed:', err.message);
    return new Response('Invalid signature', { status: 400 });
  }

  // 2. Idempotency check (prevents race conditions)
  const supabase = await SupabaseClientFactory.createAdminClient(env);
  const { data: inserted, error } = await supabase
    .from('webhook_events')
    .insert({
      stripe_event_id: event.id,
      event_type: event.type,
      received_at: new Date().toISOString()
    })
    .select('id')
    .single();

  if (error && error.code === '23505') {
    // Unique constraint violation = already processed
    console.log(`Webhook ${event.id} already processed`);
    return new Response('OK', { status: 200 });
  }

  if (error) {
    console.error('Error inserting webhook event:', error);
    throw error;
  }

  // 3. Queue for async processing (don't block Stripe)
  await env.STRIPE_WEBHOOKS_QUEUE.send({
    event_id: event.id,
    event_type: event.type,
    event_data: event.data
  });

  // 4. Return 200 immediately
  return new Response('OK', { status: 200 });
}
```

**Supabase Migration:**
```sql
-- webhook_events table with unique constraint
CREATE TABLE webhook_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stripe_event_id TEXT UNIQUE NOT NULL,
  event_type TEXT NOT NULL,
  received_at TIMESTAMPTZ NOT NULL,
  processed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_webhook_events_stripe_event_id ON webhook_events(stripe_event_id);
CREATE INDEX idx_webhook_events_event_type ON webhook_events(event_type);
```

---

## SECTION 9: CRON JOBS

### **Monthly Credit Renewal:**
```typescript
// core/cron/monthly-renewal.job.ts
export async function monthlyRenewalJob(env: Env) {
  const startTime = Date.now();
  let processed = 0;
  let succeeded = 0;
  let failed = 0;

  try {
    const supabase = await SupabaseClientFactory.createAdminClient(env);

    // Get subscriptions needing renewal
    const { data: subscriptions, error } = await supabase
      .rpc('get_renewable_subscriptions');

    if (error) throw error;

    for (const subscription of subscriptions) {
      try {
        // Idempotency check: Has this subscription been renewed in past 23 hours?
        const { data: recentRenewal } = await supabase
          .from('credit_ledger')
          .select('id')
          .eq('account_id', subscription.account_id)
          .eq('operation_type', 'subscription_renewal')
          .gte('created_at', new Date(Date.now() - 23 * 60 * 60 * 1000).toISOString())
          .limit(1)
          .single();

        if (recentRenewal) {
          console.log(`Subscription ${subscription.id} already renewed in past 23 hours`);
          processed++;
          succeeded++;
          continue;
        }

        // Grant credits (positive amount)
        await supabase.rpc('deduct_credits', {
          p_account_id: subscription.account_id,
          p_amount: subscription.credits_per_period,
          p_operation_type: 'subscription_renewal',
          p_description: `Monthly renewal: ${subscription.plan_name}`
        });

        // Update subscription period
        const newPeriodStart = new Date(subscription.current_period_end);
        const newPeriodEnd = new Date(newPeriodStart);
        newPeriodEnd.setMonth(newPeriodEnd.getMonth() + 1);

        await supabase
          .from('subscriptions')
          .update({
            current_period_start: newPeriodStart.toISOString(),
            current_period_end: newPeriodEnd.toISOString()
          })
          .eq('id', subscription.id);

        succeeded++;
      } catch (error: any) {
        console.error(`Failed to renew subscription ${subscription.id}:`, error);
        failed++;
      }

      processed++;
    }

    // Track metrics
    await env.ANALYTICS_ENGINE.writeDataPoint({
      blobs: ['monthly_renewal', 'success'],
      doubles: [processed, succeeded, failed, Date.now() - startTime],
      indexes: []
    });

    console.log(`Monthly renewal complete: ${succeeded}/${processed} succeeded, ${failed} failed`);

  } catch (error: any) {
    console.error('Monthly renewal job failed:', error);
    
    // Log to Sentry (critical error)
    // await Sentry.captureException(error);

    throw error;
  }
}
```

### **Daily Cleanup:**
```typescript
// core/cron/daily-cleanup.job.ts
export async function dailyCleanupJob(env: Env) {
  const startTime = Date.now();
  const deletedCounts: Record<string, number> = {};

  try {
    const supabase = await SupabaseClientFactory.createAdminClient(env);

    // 1. Delete old AI usage logs (90 days)
    const { count: aiLogsDeleted } = await supabase
      .from('ai_usage_logs')
      .delete()
      .lt('created_at', new Date(Date.now() - 90 * 24 * 60 * 60 * 1000).toISOString());

    deletedCounts.ai_usage_logs = aiLogsDeleted || 0;

    // 2. Hard-delete soft-deleted records (30 days old)
    const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString();

    // Delete in order (respect foreign keys)
    const { count: analysesDeleted } = await supabase
      .from('analyses')
      .delete()
      .not('deleted_at', 'is', null)
      .lt('deleted_at', thirtyDaysAgo);

    deletedCounts.analyses = analysesDeleted || 0;

    const { count: leadsDeleted } = await supabase
      .from('leads')
      .delete()
      .not('deleted_at', 'is', null)
      .lt('deleted_at', thirtyDaysAgo);

    deletedCounts.leads = leadsDeleted || 0;

    const { count: businessProfilesDeleted } = await supabase
      .from('business_profiles')
      .delete()
      .not('deleted_at', 'is', null)
      .lt('deleted_at', thirtyDaysAgo);

    deletedCounts.business_profiles = businessProfilesDeleted || 0;

    const { count: accountsDeleted } = await supabase
      .from('accounts')
      .delete()
      .not('deleted_at', 'is', null)
      .lt('deleted_at', thirtyDaysAgo);

    deletedCounts.accounts = accountsDeleted || 0;

    // Track metrics
    await env.ANALYTICS_ENGINE.writeDataPoint({
      blobs: ['daily_cleanup', 'success'],
      doubles: [
        deletedCounts.ai_usage_logs,
        deletedCounts.analyses,
        deletedCounts.leads,
        deletedCounts.business_profiles,
        deletedCounts.accounts,
        Date.now() - startTime
      ],
      indexes: []
    });

    console.log('Daily cleanup complete:', deletedCounts);

  } catch (error: any) {
    console.error('Daily cleanup job failed:', error);
    throw error;
  }
}
```

### **Failed Analysis Cleanup:**
```typescript
// core/cron/failed-analysis-cleanup.job.ts
export async function failedAnalysisCleanupJob(env: Env) {
  const startTime = Date.now();
  let refunded = 0;

  try {
    const supabase = await SupabaseClientFactory.createAdminClient(env);
    const fiveMinutesAgo = new Date(Date.now() - 5 * 60 * 1000).toISOString();

    // Find stuck analyses
    const { data: stuckAnalyses, error } = await supabase
      .from('analyses')
      .select('id, account_id, analysis_type, credits_used')
      .in('status', ['pending', 'processing'])
      .lt('created_at', fiveMinutesAgo);

    if (error) throw error;

    for (const analysis of stuckAnalyses || []) {
      try {
        // Double-check status (could have completed between query and processing)
        const { data: current } = await supabase
          .from('analyses')
          .select('status')
          .eq('id', analysis.id)
          .single();

        if (current && ['complete', 'failed', 'cancelled'].includes(current.status)) {
          continue; // Already resolved
        }

        // Mark as failed
        await supabase
          .from('analyses')
          .update({
            status: 'failed',
            error_message: 'Analysis timed out after 5 minutes'
          })
          .eq('id', analysis.id);

        // Refund credits
        await supabase.rpc('deduct_credits', {
          p_account_id: analysis.account_id,
          p_amount: -analysis.credits_used, // Negative = refund
          p_operation_type: 'timeout_refund',
          p_description: `Auto-refund for timed out ${analysis.analysis_type} analysis`
        });

        refunded++;
      } catch (error: any) {
        console.error(`Failed to refund analysis ${analysis.id}:`, error);
      }
    }

    // Track metrics
    await env.ANALYTICS_ENGINE.writeDataPoint({
      blobs: ['failed_analysis_cleanup', 'success'],
      doubles: [stuckAnalyses?.length || 0, refunded, Date.now() - startTime],
      indexes: []
    });

    console.log(`Failed analysis cleanup: ${refunded}/${stuckAnalyses?.length || 0} refunded`);

  } catch (error: any) {
    console.error('Failed analysis cleanup job failed:', error);
    throw error;
  }
}
```

---

## SECTION 10: MAIN ENTRY POINT

### **Worker Entry Point:**
```typescript
// src/index.ts
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { monthlyRenewalJob } from '@/core/cron/monthly-renewal.job';
import { dailyCleanupJob } from '@/core/cron/daily-cleanup.job';
import { failedAnalysisCleanupJob } from '@/core/cron/failed-analysis-cleanup.job';

const app = new Hono<{ Bindings: Env }>();

app.use('/*', cors());

// Health check
app.get('/health', async (c) => {
  return c.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    bindings: {
      kv: !!c.env.OSLIRA_KV,
      r2: !!c.env.R2_CACHE_BUCKET,
      workflows: !!c.env.ANALYSIS_WORKFLOW,
      queues: !!c.env.ANALYSIS_QUEUE,
      durable_objects: !!c.env.ANALYSIS_PROGRESS,
      analytics: !!c.env.ANALYTICS_ENGINE
    }
  });
});

// Progress endpoint
app.get('/runs/:run_id/progress', async (c) => {
  const run_id = c.req.param('run_id');

  try {
    const doId = c.env.ANALYSIS_PROGRESS.idFromName(run_id);
    const progressTracker = c.env.ANALYSIS_PROGRESS.get(doId);
    const response = await progressTracker.fetch('https://fake-host/progress');
    return response;
  } catch (error) {
    // Fallback to database
    const supabase = await SupabaseClientFactory.createUserClient(c.env);
    const { data: run } = await supabase
      .from('runs')
      .select('status, created_at')
      .eq('run_id', run_id)
      .single();

    return c.json({
      run_id,
      status: run?.status || 'unknown',
      progress: run?.status === 'complete' ? 100 : 50,
      current_step: 'Processing...',
      fallback: true
    });
  }
});

// Cancel endpoint
app.post('/runs/:run_id/cancel', async (c) => {
  const run_id = c.req.param('run_id');
  
  const doId = c.env.ANALYSIS_PROGRESS.idFromName(run_id);
  const progressTracker = c.env.ANALYSIS_PROGRESS.get(doId);
  
  await progressTracker.fetch('https://fake-host/cancel', {
    method: 'POST'
  });
  
  return c.json({ success: true, message: 'Analysis cancellation requested' });
});

// Cron handler
export default {
  fetch: app.fetch,
  
  async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext) {
    const cron = event.cron;
    
    try {
      if (cron === '0 3 1 * *') {
        // Monthly renewal (1st of month, 3 AM UTC)
        await monthlyRenewalJob(env);
      } else if (cron === '0 2 * * *') {
        // Daily cleanup (2 AM UTC)
        await dailyCleanupJob(env);
      } else if (cron === '0 * * * *') {
        // Hourly failed analysis cleanup
        await failedAnalysisCleanupJob(env);
      }
    } catch (error) {
      console.error('Cron job failed:', error);
      // Log to Sentry
      // await Sentry.captureException(error);
    }
  }
};

// Export Durable Object
export { AnalysisProgressTracker } from '@/core/durable-objects/analysis-progress.do';
```

---

## SECTION 11: RLS MIGRATION

### **Supabase RLS Fix (Apply to ALL policies):**
```sql
-- BEFORE (SLOW - 450ms)
CREATE POLICY "users_select_own_leads"
ON leads FOR SELECT
USING (account_id = auth.uid());

-- AFTER (FAST - 4ms)
CREATE POLICY "users_select_own_leads"
ON leads FOR SELECT
USING (account_id = (SELECT auth.uid()));

-- Apply to ALL tables with RLS:
-- leads, analyses, business_profiles, credit_balances, credit_transactions, subscriptions

-- Example: Update all existing policies
DROP POLICY IF EXISTS "users_select_own_leads" ON leads;
CREATE POLICY "users_select_own_leads"
ON leads FOR SELECT
USING (account_id = (SELECT auth.uid()));

DROP POLICY IF EXISTS "users_insert_own_leads" ON leads;
CREATE POLICY "users_insert_own_leads"
ON leads FOR INSERT
WITH CHECK (account_id = (SELECT auth.uid()));

DROP POLICY IF EXISTS "users_update_own_leads" ON leads;
CREATE POLICY "users_update_own_leads"
ON leads FOR UPDATE
USING (account_id = (SELECT auth.uid()));

DROP POLICY IF EXISTS "users_delete_own_leads" ON leads;
CREATE POLICY "users_delete_own_leads"
ON leads FOR DELETE
USING (account_id = (SELECT auth.uid()));

-- Repeat for: analyses, business_profiles, credit_balances, credit_transactions, subscriptions
```

