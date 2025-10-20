# 🔍 MISSING CRITICAL LOGIC FROM CONSOLIDATED

Here's everything you're missing that's NOT just "how to write code" but actual **business logic, execution flow, and implementation decisions**:

---

## ❌ 1. ANALYSIS EXECUTION FLOW (Critical Gap)

### **Current State: Undefined**
Your consolidated says "analyze profile" but doesn't specify:

### **Missing: Exact Step Order**
```
1. Receive request → Generate run_id
2. Create pending record in runs table
3. ✅ CHECK: Duplicate analysis already running? (MISSING)
4. ✅ CHECK: Sufficient credits? (MISSING)
5. Deduct credits IMMEDIATELY (atomic, prevents race conditions)
6. Check R2 cache for profile
   - Cache hit → Skip to step 9
   - Cache miss → Continue to step 7
7. Scrape profile via Apify (6-8s)
8. Store scraped profile in R2 cache
9. Run AI analysis (parallel calls)
10. Save results to database
11. Update run status to 'complete'
12. Update Durable Object progress (if used)
```

### **Missing: Duplicate Analysis Prevention**
**Problem:** User clicks "Analyze" twice within 2 seconds → 2 analyses run → double credit deduction

**Solution:**
```sql
-- Before deducting credits, check:
SELECT run_id FROM analyses 
WHERE username = $1 
  AND account_id = $2 
  AND status IN ('pending', 'processing')
  AND created_at > NOW() - INTERVAL '5 minutes'
LIMIT 1;

-- If exists → Return error: "Analysis already in progress for this profile"
-- If not exists → Proceed with credit deduction
```

### **Missing: Credit Deduction Timing Decision**
**Two Options:**

**Option A: Deduct BEFORE scraping (RECOMMENDED)**
- ✅ Prevents race conditions
- ✅ User committed to purchase
- ❌ If Apify fails → Must refund credits

**Option B: Deduct AFTER scraping**
- ✅ Only charge if scraping succeeds
- ❌ Race condition window (user could spam requests)
- ❌ More complex state management

**Your doc doesn't specify which. Recommendation: Option A with auto-refund on failure.**

---

## ❌ 2. R2 CACHE STRATEGY (Incomplete)

### **Missing: Cache Invalidation Logic**
Your doc says "cache profiles" but doesn't define:

**When to invalidate cache:**
- User manually refreshes profile (bypass cache)
- Profile follower count changed >10% since cached
- Profile bio changed
- Cache older than TTL

**How to handle partial cache:**
```typescript
// Cached profile has posts but missing stories
// Do we:
// A) Use cached posts, scrape only stories
// B) Invalidate entire cache, scrape everything
// C) Store posts/stories separately in cache

// RECOMMENDATION: B - Simpler, profile data changes together
```

### **Missing: Cache Key Versioning**
```typescript
// Current: instagram:nike:v1
// What if scraping logic changes?
// Need: instagram:nike:v2 (bump on breaking changes)

// How to handle version migration:
// - Keep v1 cache alive for 7 days
// - New analyses use v2
// - Background job deletes v1 after 7 days
```

### **Missing: Cache Storage Limits**
**R2 Pricing:** $0.015/GB-month storage

**Calculation:**
- Average profile: 50KB (JSON)
- 10,000 cached profiles: 500MB
- Cost: $0.0075/month (negligible)
- **Conclusion:** No storage limits needed, cache everything

---

## ❌ 3. PROMPT CACHING EXACT IMPLEMENTATION

### **Missing: Business Context Structure**
Your doc says "cache business context" but doesn't show:

**What exactly gets cached:**
```typescript
// Business context (800 tokens - CACHED)
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

// Profile data (NEVER cached - changes per request)
const profileData = `
Username: @${profile.username}
Followers: ${profile.followersCount}
Bio: ${profile.biography}
Recent Posts: [${profile.posts.length} posts analyzed]
...
`;
```

### **Missing: Cache Hit Rate Optimization**
**Problem:** If business profile changes, cache misses increase

**Solution:**
```typescript
// Include business_profile version in cache key
const cacheKey = `business:${business.id}:v${business.updated_at.getTime()}`;

// OR: Hash business context
const contextHash = hash(businessContext);
const cacheKey = `business:${business.id}:${contextHash}`;
```

---

## ❌ 4. AI MODEL SELECTION REASONING

### **Missing: Why These Specific Models**

**LIGHT: gpt-4o-mini**
- ✅ Fastest (2-3s)
- ✅ Cheapest ($0.15/1M input tokens)
- ✅ Good at JSON structured output
- ❌ Limited reasoning depth (fine for simple scoring)

**DEEP core: gpt-5-mini**
- ✅ Balanced speed/quality (12-15s)
- ✅ Better reasoning than gpt-4o-mini
- ❌ More expensive than gpt-4o-mini
- **Why not gpt-4o-mini:** Core strategy needs deeper analysis
- **Why not gpt-5:** Overkill, 2x cost for marginal gain

**XRAY psychographic: gpt-5**
- ✅ Deepest psychological insight
- ✅ Best at nuanced personality analysis
- ❌ Slowest (7-10s)
- ❌ Most expensive ($0.60/1M input tokens)
- **Why worth it:** Psychographic analysis is premium feature, users expect depth

**XRAY commercial/outreach: gpt-5-mini**
- ✅ Sufficient for tactical recommendations
- ❌ gpt-5 overkill for these tasks

### **Missing: Reasoning Effort Optimization**
```typescript
// OpenAI reasoning effort levels:
// - low: Fast, less thorough
// - medium: Balanced
// - high: Slowest, most thorough

const REASONING_CONFIG = {
  LIGHT: {
    scoring: 'low'  // Simple numeric scoring
  },
  DEEP: {
    core_strategy: 'medium',  // Needs depth but not max
    outreach: 'low',          // Tactical, template-based
    personality: 'low'        // Pattern recognition
  },
  XRAY: {
    psychographic: 'medium',  // Deep but not exhaustive
    commercial: 'medium',     // Strategic decisions
    outreach: 'low',
    personality: 'low'
  }
};

// Why not 'high' anywhere?
// - 'high' adds 10-20s per call
// - Quality gain: ~5%
// - Not worth latency for user experience
```

---

## ❌ 5. BULK ANALYSIS EXACT LOGIC

### **Missing: Apify Concurrency Handling**

**Problem:** Apify Starter plan = 10 concurrent actors max

**Your Scenario: 100 profiles**
```typescript
// ❌ WRONG: Start all 100 at once → Apify rejects 90
Promise.all(usernames.map(u => scrapeProfile(u)));

// ✅ CORRECT: Batch into groups of 10
const batches = chunkArray(usernames, 10);
// batches = [ [1-10], [11-20], [21-30], ... [91-100] ]

// Process sequentially (each batch waits for previous)
for (const batch of batches) {
  await Promise.all(batch.map(u => scrapeProfile(u)));
  // Batch 1: 10 profiles (8s)
  // Batch 2: 10 profiles (8s)
  // ...
  // Total: 10 batches × 8s = 80s
}
```

### **Missing: Partial Failure Handling**
```typescript
// Batch 1: 10 profiles
// - 9 succeed
// - 1 fails (Apify timeout)

// What happens?
// Option A: Fail entire batch → Refund all 10 credits
// Option B: Continue, only refund 1 credit
// Option C: Retry failed profile 3x before refunding

// RECOMMENDATION: Option C
// - Retry with exponential backoff (1s, 2s, 4s)
// - If all retries fail → Mark as failed, refund 1 credit
// - Continue processing remaining profiles
```

### **Missing: Progress Granularity**
```typescript
// User sees: "Processing 100 profiles..."
// But what detail level?

// Option A: Simple
{ completed: 47, total: 100, progress: 47% }

// Option B: Detailed (RECOMMENDED)
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

## ❌ 6. COST TRACKING FORMULAS

### **Missing: Exact Apify Cost Calculation**

**Apify Pricing:**
- $0.25 per compute unit (CU)
- 1 CU = 1 hour of scraping
- Instagram Profile Scraper: ~0.01 CU per profile (36s avg)

**Formula:**
```typescript
function calculateApifyCost(posts: number, durationMs: number): number {
  const computeUnits = durationMs / (1000 * 60 * 60); // Convert ms to hours
  const cost = computeUnits * 0.25;
  return cost;
}

// Example:
// 50 posts, 8 seconds
// = 8000ms / 3600000 = 0.00222 CU
// = 0.00222 × $0.25 = $0.00056
```

### **Missing: AI Cost Per Model**
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

### **Missing: Margin Calculation**
```typescript
// LIGHT: 1 credit = $0.97 revenue
// Costs: Apify ($0.0006) + AI ($0.0003) = $0.0009
// Margin: $0.97 - $0.0009 = $0.9691 (99.9% margin)

// DEEP: 5 credits = $4.85 revenue
// Costs: Apify ($0.0006) + AI ($0.015) = $0.0156
// Margin: $4.85 - $0.0156 = $4.8344 (99.7% margin)

// XRAY: 6 credits = $5.82 revenue
// Costs: Apify ($0.0006) + AI ($0.025) = $0.0256
// Margin: $5.82 - $0.0256 = $5.7944 (99.6% margin)
```

---

## ❌ 7. DURABLE OBJECT LIFECYCLE

### **Missing: Creation → Hibernation → Cleanup Flow**

```typescript
// 1. CREATION (when analysis starts)
POST /v1/analyze
  ↓
Create run_id
  ↓
Create Durable Object: ANALYSIS_PROGRESS.idFromName(run_id)
  ↓
Initialize state: { run_id, status: 'queued', progress: 0 }

// 2. ACTIVE PHASE (during analysis)
Workflow updates DO every step:
  - "Checking cache" (10%)
  - "Scraping profile" (30%)
  - "Running AI analysis" (70%)
  - "Saving results" (90%)
  - "Complete" (100%)

Client polls DO every 2s:
  GET /runs/:run_id/progress

// 3. COMPLETION
Analysis finishes
  ↓
Workflow sends final update: { status: 'complete', progress: 100 }
  ↓
DO sets alarm: ctx.storage.setAlarm(Date.now() + 3600000) // 1 hour
  ↓
Client stops polling (sees status: 'complete')

// 4. CLEANUP (1 hour later)
Alarm fires
  ↓
DO deletes its own state: ctx.storage.deleteAll()
  ↓
DO becomes inactive (Cloudflare GC's it)
```

### **Missing: Hibernation Decision**
**Hibernation = DO goes to sleep between requests, wakes on next request**

**Should you enable it?**
- ✅ Enable if: DO idle >10 seconds between updates (reduces costs)
- ❌ Disable if: High-frequency updates (analysis < 30s total)

**For OSLIRA:**
- LIGHT: 6-11s analysis → ❌ No hibernation (too fast)
- DEEP: 18-23s analysis → ❌ No hibernation (frequent updates)
- XRAY: 16-18s analysis → ❌ No hibernation (frequent updates)

**Conclusion:** Disable hibernation, analyses too short to benefit

### **Missing: Fallback Strategy**
```typescript
// What if Durable Object unavailable?
// (Cloudflare maintenance, regional outage)

// Option A: Fail entire request → Bad UX
// Option B: Store progress in database → Slow (extra writes)
// Option C: Return basic status from database → RECOMMENDED

async function getAnalysisProgress(run_id: string, env: Env) {
  try {
    // Try DO first
    const doId = env.ANALYSIS_PROGRESS.idFromName(run_id);
    const progress = await doId.fetch('/progress');
    return await progress.json();
  } catch (error) {
    // Fallback to database
    const run = await db.query(
      'SELECT status, created_at FROM runs WHERE run_id = $1',
      [run_id]
    );
    
    return {
      run_id,
      status: run.status,
      progress: run.status === 'complete' ? 100 : 50,
      current_step: 'Processing...',
      fallback: true // Flag to indicate degraded mode
    };
  }
}
```

---

## ❌ 8. WEBHOOK IDEMPOTENCY EXACT FLOW

### **Missing: Race Condition Handling**

**Problem:** 2 webhook requests arrive simultaneously

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

-- If RETURNING id is NULL → Another request already inserted
-- If RETURNING id has value → We won the race, process event
```

**Full Flow:**
```typescript
export async function handleStripeWebhook(request: Request, env: Env) {
  // 1. Verify signature
  const signature = request.headers.get('stripe-signature');
  const payload = await request.text();
  
  let event;
  try {
    event = stripe.webhooks.constructEvent(payload, signature, webhookSecret);
  } catch (err) {
    return new Response('Invalid signature', { status: 400 });
  }
  
  // 2. Check if already processed (idempotency)
  const result = await db.query(`
    INSERT INTO webhook_events (stripe_event_id, event_type, received_at)
    VALUES ($1, $2, NOW())
    ON CONFLICT (stripe_event_id) DO NOTHING
    RETURNING id
  `, [event.id, event.type]);
  
  if (!result.rows[0]) {
    // Already processed by another request
    console.log(`Webhook ${event.id} already processed`);
    return new Response('OK', { status: 200 }); // Return 200 to Stripe
  }
  
  // 3. Queue for processing
  await env.STRIPE_WEBHOOKS_QUEUE.send({
    event_id: event.id,
    event_type: event.type,
    event_data: event.data
  });
  
  // 4. Return 200 immediately (don't make Stripe wait)
  return new Response('OK', { status: 200 });
}
```

---

## ❌ 9. LEAD UPSERT EXACT LOGIC

### **Missing: Field Update Priority**

**Always Update:**
- `follower_count` (changes frequently)
- `bio` (users change often)
- `profile_pic_url` (users change)
- `last_analyzed_at` (timestamp)

**Never Update:**
- `first_analyzed_at` (historical record)
- `created_at` (audit trail)
- `account_id`, `business_profile_id` (foreign keys)

**Conditionally Update:**
- `instagram_url` → Only if NULL (URL doesn't change)
- `is_verified` → Always update (status can change)

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
-- xmax = 0 means INSERT happened (new row)
-- xmax != 0 means UPDATE happened (existing row)
```

---

## ❌ 10. CRON JOB IDEMPOTENCY

### **Missing: Monthly Renewal Deduplication**

**Problem:** Cron runs twice (Cloudflare retry) → Double credit grant

**Solution: Check credit_ledger for recent grant**
```sql
-- Before granting credits, check:
SELECT id FROM credit_ledger
WHERE account_id = $1
  AND operation_type = 'subscription_renewal'
  AND created_at > NOW() - INTERVAL '23 hours'
LIMIT 1;

-- If exists → Skip (already renewed in past 23 hours)
-- If not exists → Grant credits
```

### **Missing: Failed Analysis Timeout Logic**

**Decision: What timeout threshold?**
- LIGHT: Should complete in 11s, fail after 5 min
- DEEP: Should complete in 23s, fail after 5 min
- XRAY: Should complete in 18s, fail after 5 min

**Reasoning:** 5 minutes = ~13x expected time, catches stuck analyses

```sql
UPDATE analyses
SET status = 'failed',
    error_message = 'Analysis timed out after 5 minutes'
WHERE status IN ('pending', 'processing')
  AND created_at < NOW() - INTERVAL '5 minutes'
RETURNING id, account_id, credits_used;

-- Then refund credits
FOREACH failed_analysis IN returned_analyses
  CALL deduct_credits(
    failed_analysis.account_id,
    -failed_analysis.credits_used, -- Negative = refund
    'timeout_refund',
    'Auto-refund for timed out analysis'
  );
```

---

## ❌ 11. RLS PERFORMANCE FIX IMPACT

### **Missing: Before/After Metrics**

**Before (Slow):**
```sql
CREATE POLICY "Users see own leads"
ON leads FOR SELECT
USING (account_id = auth.uid());

-- Query plan:
-- Seq Scan on leads (cost=0.00..1234.56)
--   Filter: (account_id = auth.uid())
-- Execution time: 450ms (10,000 rows scanned)
```

**After (Fast):**
```sql
CREATE POLICY "Users see own leads"
ON leads FOR SELECT
USING (account_id = (SELECT auth.uid()));

-- Query plan:
-- Index Scan using leads_account_id_idx (cost=0.29..8.31)
--   Index Cond: (account_id = $1)
-- Execution time: 4ms (index seek)
```

**Why it works:**
- `auth.uid()` re-executes for every row (expensive function call)
- `(SELECT auth.uid())` executes once, result cached in query plan
- 100x faster confirmed by Supabase engineering

### **Missing: Which Tables to Apply**

**Apply to ALL policies on:**
- ✅ `leads` (largest table, most queries)
- ✅ `analyses` (queried frequently)
- ✅ `business_profiles`
- ✅ `credit_balances`
- ✅ `credit_transactions`
- ✅ `subscriptions`
- ❌ `accounts` (no RLS, admin-only access)

---

## ❌ 12. ASYNC EXECUTION USER EXPERIENCE

### **Missing: Polling vs WebSocket Decision**

**WebSocket:**
- ✅ Real-time (instant updates)
- ❌ Complex (connection management)
- ❌ Scaling issues (many open connections)

**HTTP Polling:**
- ✅ Simple (standard REST)
- ✅ Scales well (stateless)
- ❌ 2-second delay (acceptable for 10-30s analyses)

**Decision: HTTP polling (simpler, good enough)**

### **Missing: Client-Side Polling Pattern**
```typescript
// Frontend
async function startAnalysis(username: string) {
  // 1. Start analysis
  const response = await fetch('/v1/analyze', {
    method: 'POST',
    body: JSON.stringify({ username, analysis_type: 'deep' })
  });
  
  const { run_id } = await response.json();
  
  // 2. Poll for progress
  const interval = setInterval(async () => {
    const progress = await fetch(`/runs/${run_id}/progress`);
    const data = await progress.json();
    
    // Update UI
    updateProgressBar(data.progress);
    updateStatus(data.current_step);
    
    // Stop polling when done
    if (['complete', 'failed', 'cancelled'].includes(data.status)) {
      clearInterval(interval);
      
      if (data.status === 'complete') {
        showResults(data.result);
      } else {
        showError(data.error_message);
      }
    }
  }, 2000); // Poll every 2 seconds
}
```

### **Missing: Stop Polling Trigger**
```typescript
// Don't poll forever if user navigates away
useEffect(() => {
  const interval = startPolling(run_id);
  
  return () => {
    clearInterval(interval); // Cleanup on unmount
  };
}, [run_id]);
```

---

## ✅ SUMMARY: ADD THESE TO YOUR CONSOLIDATED

1. **Analysis execution step order** (12 steps with decision points)
2. **Duplicate analysis prevention** (SQL check before deduction)
3. **Credit deduction timing** (before scraping, with refund on failure)
4. **Cache invalidation rules** (TTL + manual refresh)
5. **Cache key versioning strategy** (v1 → v2 migration)
6. **Business context exact structure** (800 tokens cached)
7. **AI model selection reasoning** (why each model for each task)
8. **Reasoning effort optimization** (low/medium/high per task)
9. **Bulk analysis Apify batching** (10 concurrent, sequential batches)
10. **Partial failure handling** (retry 3x, refund if all fail)
11. **Apify cost formula** (CU calculation)
12. **AI cost formula** (per token pricing)
13. **Margin calculation** (revenue - costs per analysis type)
14. **Durable Object lifecycle** (creation → active → cleanup)
15. **DO hibernation decision** (disabled for short analyses)
16. **DO fallback strategy** (database if DO unavailable)
17. **Webhook race condition handling** (ON CONFLICT DO NOTHING)
18. **Lead upsert field priority** (what to always/never/conditionally update)
19. **Cron idempotency checks** (credit_ledger check)
20. **Failed analysis timeout threshold** (5 minutes)
21. **RLS fix performance impact** (450ms → 4ms)
22. **Polling vs WebSocket decision** (polling chosen)
23. **Client-side polling pattern** (2s interval, stop on complete)

**This is your missing "execution logic layer" - add to consolidated.**
