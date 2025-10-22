# OSLIRA CLOUDFLARE WORKER - IMPLEMENTATION SPEC v6.0

**Status:** Production Ready | **Implementation Time:** 10-11 Weeks

---

## 📊 CURRENT STATE vs TARGET STATE

### Current Worker: 3/10

**Critical Failures:**
- ❌ Synchronous blocking (30s timeout kills analyses)
- ❌ Service role everywhere (RLS bypassed = security risk)
- ❌ Zero observability (no cost/performance tracking)
- ❌ No progress tracking (refresh = lost state)
- ❌ No retry logic (Apify fail = lost credits)
- ❌ Mixed architecture (40+ scattered files)
- ❌ Race conditions (duplicate credit deductions)
- ❌ user_id instead of account_id (wrong multi-tenant)
- ❌ No duplicate analysis prevention (spam = double charge)

**What Works:**
- ✅ AWS Secrets integration
- ✅ Path aliases configured
- ✅ Domain folders exist

---

### Target Worker: 97/100

**Core Infrastructure:**
- ✅ Workflows = async orchestration (no timeout)
- ✅ Dual Supabase client (RLS enforced + admin)
- ✅ Cloudflare AI Gateway (30-40% cost savings, FREE)
- ✅ Durable Objects (real-time progress + cancel)
- ✅ Feature-first architecture (vertical slices)
- ✅ Comprehensive observability (Analytics + Sentry)
- ✅ Cloudflare Cron (renewals, cleanup)

**Performance Gains:**
- LIGHT: 10-13s → 6s avg (50% cache hit)
- DEEP: 30s → 18-23s cold, 12-15s cached
- XRAY: 35s → 16-18s cold, 10-12s cached
- Bulk 100: 16min → 60-120s

**Cost Reductions:**
- AI: $300/mo → $210/mo (prompt caching + AI Gateway)
- Worker CPU: $100/mo → $2/mo (Workflows)
- **Total savings: $188/month**

**Why not 100/100?** Needs production battle-testing.

---

## 🎯 ARCHITECTURE

```
src/
├── features/              # Vertical slices
│   ├── credits/          # domain/application/controllers/routes.ts
│   ├── auth/
│   ├── business/
│   └── analysis/
├── core/                  # Horizontal services
│   ├── workflows/        # deep/light/xray/bulk-analysis.workflow.ts + steps/
│   ├── queues/           # stripe-webhook.consumer, analysis.consumer
│   ├── durable-objects/  # analysis-progress.do.ts
│   └── cron/             # monthly-renewal, daily-cleanup, failed-cleanup
├── infrastructure/        # External adapters
│   ├── database/         # supabase.client + repositories/
│   ├── ai/               # openai.adapter, claude.adapter, ai-gateway.client
│   ├── scraping/         # apify.adapter
│   ├── cache/            # r2-cache.service
│   └── monitoring/       # analytics-engine, sentry, cost/performance trackers
└── shared/               # middleware/, utils/, types/
```

---

## 🔧 12-STEP ANALYSIS FLOW

```
1. Generate run_id
2. Create pending record in runs table
3. CHECK: Duplicate in progress? (5min window) → Error if exists
4. CHECK: Sufficient credits? → Error if insufficient
5. DEDUCT CREDITS IMMEDIATELY (atomic RPC) → Refund if later steps fail
6. Check R2 cache (key: instagram:username:v1) → Skip to 9 if hit
7. Scrape via Apify (6-8s) with retry
8. Store in R2 cache (TTL: 24h/12h/6h by type)
9. Run AI analysis (parallel for DEEP/XRAY, business context cached)
10. UPSERT lead (username+account_id+business_id+deleted_at=NULL)
    - If exists: Update follower_count, bio, profile_pic_url, last_analyzed_at
    - If not: Insert new
11. Save analysis results
12. Update status='complete', progress=100%
```

**Decision: Credit deduction BEFORE scraping (Step 5)**
- ✅ Prevents race conditions (user committed)
- ✅ No spam clicks
- ❌ Must refund on failure (acceptable)

---

## 🚨 CRITICAL REQUIREMENTS

### **MUST IMPLEMENT:**

1. **Dual Supabase Client**
   - RLS client (user operations) - `createRLSClient(request, env)`
   - Admin client (system ops) - `createAdminClient(env)`
   - **RULE:** Use RLS by default. Admin ONLY for: cron jobs, webhooks, on-behalf-of operations

2. **Atomic Credit Deduction (RPC)**
   ```sql
   CREATE FUNCTION deduct_credits_atomic(
     p_account_id UUID, p_amount INT, 
     p_analysis_type TEXT, p_run_id UUID
   ) RETURNS JSON AS $$
   BEGIN
     -- Lock row FOR UPDATE
     -- Check balance >= amount
     -- Update balance
     -- Insert credit_transactions
     -- Return new_balance, transaction_id
   END;
   ```

3. **Lead Upsert (NOT Insert)**
   ```sql
   INSERT INTO leads (username, account_id, business_profile_id, ...)
   VALUES (...)
   ON CONFLICT (username, account_id, business_profile_id)
   WHERE deleted_at IS NULL
   DO UPDATE SET
     follower_count = EXCLUDED.follower_count,
     bio = EXCLUDED.bio,
     last_analyzed_at = NOW()
   RETURNING id, (xmax = 0) AS is_new;
   ```

4. **RLS Performance Fix (100x faster)**
   ```sql
   -- BEFORE (450ms): account_id = auth.uid()
   -- AFTER (4ms): account_id = (SELECT auth.uid())
   -- Apply to ALL policies on: leads, analyses, business_profiles, 
   -- credit_balances, credit_transactions, subscriptions
   ```

5. **Webhook Idempotency**
   ```sql
   INSERT INTO webhook_events (stripe_event_id, event_type, received_at)
   VALUES ($1, $2, NOW())
   ON CONFLICT (stripe_event_id) DO NOTHING
   RETURNING id;
   -- If id IS NULL → Already processed (lost race)
   ```

6. **R2 Cache Strategy**
   - Key: `instagram:${username}:v1`
   - TTL: 24h (LIGHT), 12h (DEEP), 6h (XRAY)
   - Invalidate if: TTL expired, follower_count changed >10%, bio changed significantly

7. **Prompt Caching**
   - Business context (800 tokens) = CACHED (changes rarely)
   - Profile data (dynamic) = NOT CACHED
   - OpenAI auto-caches after 1024 tokens
   - Claude needs explicit `cache_control: { type: 'ephemeral' }`

8. **AI Parallelization**
   ```typescript
   // DEEP: Sequential 19s → Parallel 15s
   const [core, outreach, personality] = await Promise.all([
     executeCoreStrategy(),   // 15s
     executeOutreach(),       // 4s
     executePersonality()     // 4s
   ]); // Total: max(15,4,4) = 15s
   ```

9. **Bulk Analysis Batching**
   - Apify limit: 10 concurrent actors
   - Batch 100 usernames into 10 groups of 10
   - Process sequentially (each batch waits)
   - Partial failure: Retry 3x with exponential backoff, refund if all fail

10. **Durable Object Lifecycle**
    - Creation: Generate run_id → Create DO
    - Active: Workflow updates every step (0-100%)
    - Client polls every 2s via GET /runs/:run_id/progress
    - Completion: Set cleanup alarm (1hr after complete)
    - Cleanup: DO deletes state, becomes inactive

11. **Workflow Retry Config**
    - Scraping: maxRetries=3, retryDelay=5s, exponential backoff
    - AI calls: maxRetries=2, retryDelay=3s
    - Business errors (insufficient credits): NO retry
    - Validation errors: NO retry

12. **Cron Idempotency**
    - Monthly renewal: Check credit_ledger for renewal in past 23 hours (not 24, allows clock drift)
    - Failed analysis: 5-minute timeout → Mark failed + refund

---

### **MUST NOT:**

1. ❌ **Never use service role for user-initiated requests** → Always RLS client
2. ❌ **Never use auth.uid() without subquery** → Always `(SELECT auth.uid())`
3. ❌ **Never insert leads** → Always upsert with ON CONFLICT
4. ❌ **Never deduct credits after scraping** → Always before (Step 5)
5. ❌ **Never process webhook without idempotency check** → Insert with ON CONFLICT
6. ❌ **Never grant credits without checking recent renewal** → 23-hour window
7. ❌ **Never use 'high' reasoning effort** → Adds 10-20s for only 5% quality gain
8. ❌ **Never cache profile data in prompts** → Only business context
9. ❌ **Never sequential AI calls if parallel possible** → Use Promise.all
10. ❌ **Never exceed Apify 10 concurrent limit** → Batch properly
11. ❌ **Never keep DO alive indefinitely** → Cleanup alarm after 1hr
12. ❌ **Never return 500 to Stripe webhooks** → Always 200 (even if queuing fails)

---

## 📋 IMPLEMENTATION PHASES (11 Phases, 10-11 Weeks)

### Phase 0: Pre-Setup (Week 1)
**Build:** Environment, AWS Secrets, Cloudflare bindings
- wrangler.toml: KV, R2, Workflows, Queues, DO, Cron triggers
- AWS Secrets: Supabase keys, Apify, OpenAI, Anthropic, Stripe, Sentry
- tsconfig.json: Path aliases (@/features, @/core, @/infrastructure, @/shared)
- package.json: Dependencies (hono, @supabase/supabase-js, openai, anthropic, stripe)

**Success:** `npm run dev` starts, secrets accessible

---

### Phase 1: Infrastructure Foundation (Week 1)
**Build:**
```
infrastructure/
├── database/
│   ├── supabase.client.ts         # createRLSClient + createAdminClient
│   └── repositories/
│       ├── base.repository.ts
│       ├── credits.repository.ts
│       ├── leads.repository.ts
│       └── analysis.repository.ts
├── cache/r2-cache.service.ts       # get/set with TTL checks
└── monitoring/
    ├── analytics-engine.client.ts  # Track costs, performance
    ├── cost-tracker.service.ts     # calculateApifyCost, calculateAICost
    └── performance-tracker.service.ts # Track P50/P95/P99
```

**Key Implementations:**
- Dual client factory (RLS vs admin)
- R2 cache with TTL (24h/12h/6h)
- Cost tracking formulas (Apify $0.25/CU, AI per-model pricing)

**Success:** Can create records with RLS, cache to R2, track costs

---

### Phase 2: Shared Utilities (Week 2)
**Build:**
```
shared/
├── middleware/
│   ├── auth.middleware.ts         # JWT validation, attach userId/accountId
│   ├── rate-limit.middleware.ts   # KV-based rate limiting
│   ├── error.middleware.ts        # Catch all errors, log to Sentry
│   └── analytics.middleware.ts    # Log all requests
├── utils/
│   ├── logger.util.ts             # Structured JSON logging
│   ├── validation.util.ts         # Input validation helpers
│   └── response.util.ts           # Standard response format
└── types/                          # TypeScript definitions
```

**Standard Response:**
```typescript
{ success: boolean, data?: T, error?: string, requestId: string, timestamp: string }
```

**Success:** Auth middleware works, rate limiting blocks, errors formatted

---

### Phase 3: Domain Models (Week 3)
**Build:**
```
features/analysis/domain/
├── config/analysis.config.ts       # LIGHT/DEEP/XRAY configs
├── entities/
│   ├── analysis.entity.ts
│   └── lead.entity.ts
├── value-objects/
│   ├── analysis-score.vo.ts        # 0-100 validation
│   ├── instagram-handle.vo.ts
│   └── analysis-type.vo.ts
└── services/
    ├── lead-scorer.service.ts
    └── duplicate-checker.service.ts
```

**Analysis Config:**
```typescript
LIGHT: { credit_cost: 1, price_usd: 0.97, ai_model: 'gpt-4o-mini', cache_ttl_hours: 24, posts_limit: 12 }
DEEP: { credit_cost: 5, price_usd: 4.85, parallel_calls: 3, models: {core:'gpt-5-mini', outreach:'gpt-5-mini', personality:'gpt-5-mini'}, cache_ttl_hours: 12, posts_limit: 50 }
XRAY: { credit_cost: 6, price_usd: 5.82, parallel_calls: 4, models: {psychographic:'gpt-5', commercial:'gpt-5-mini', outreach:'gpt-5-mini', personality:'gpt-5-mini'}, cache_ttl_hours: 6, posts_limit: 50 }
```

**Success:** Config centralized, entities enforce rules, value objects validated

---

### Phase 4: Credits Feature (Week 4)
**Build:**
```
features/credits/
├── domain/
│   ├── credit-amount.vo.ts
│   └── credit-calculator.service.ts
├── application/
│   ├── get-balance.usecase.ts
│   ├── get-transactions.usecase.ts
│   └── ports/credit.repository.interface.ts
├── controllers/
│   ├── balance.controller.ts       # GET /credits/balance
│   └── transactions.controller.ts  # GET /credits/transactions
└── routes.ts
```

**Key:** Validates feature-first architecture pattern

**Success:** Balance endpoint works, transactions paginated, RLS enforced

---

### Phase 5: Auth Feature (Week 4)
**Build:**
```
features/auth/
├── domain/jwt-validator.service.ts
├── application/
│   ├── verify-token.usecase.ts
│   └── ports/auth.repository.interface.ts
├── controllers/auth.controller.ts  # GET /auth/verify
└── routes.ts
```

**Success:** Valid JWT returns user, invalid JWT returns 401

---

### Phase 6: Business Profiles Feature (Week 5)
**Build:**
```
features/business/
├── domain/
│   ├── business-profile.entity.ts
│   ├── icp-matcher.service.ts
│   └── value-objects/icp-criteria.vo.ts
├── application/use-cases/
│   ├── create-profile.usecase.ts
│   ├── update-profile.usecase.ts
│   ├── get-profiles.usecase.ts
│   └── delete-profile.usecase.ts
├── controllers/profiles.controller.ts
└── routes.ts
```

**ICP Criteria:** follower_range, engagement_min, content_themes, geo_focus

**Success:** CRUD works, soft delete preserves integrity

---

### Phase 7: Analysis Feature (Weeks 5-6)
**Build:**
```
features/analysis/
├── domain/services/
│   ├── ai-analysis.service.ts      # executeCoreStrategy, generateOutreach, etc.
│   ├── prompt-builder.service.ts   # buildBusinessContext (cached), buildProfileData
│   └── result-parser.service.ts
├── application/use-cases/
│   ├── analyze-lead.usecase.ts     # Orchestrate 12-step flow
│   ├── bulk-analyze.usecase.ts
│   ├── get-analysis.usecase.ts
│   └── cancel-analysis.usecase.ts
├── controllers/
│   ├── analyze.controller.ts       # POST /v1/analyze
│   ├── bulk-analyze.controller.ts  # POST /v1/bulk-analyze
│   ├── results.controller.ts       # GET /runs/:run_id
│   └── progress.controller.ts      # GET /runs/:run_id/progress
└── routes.ts
```

**12-Step Flow Implemented Here:**
1. Generate run_id
2. Create pending record
3. Check duplicate (5min window)
4. Check credits
5. Deduct credits (atomic)
6-12. Handled by workflow (Phase 8)

**Success:** LIGHT <11s cold, DEEP <25s, XRAY <25s, cache hit >30%, no race conditions

---

### Phase 8: Async Orchestration (Weeks 7-8)
**Build:**
```
core/workflows/
├── deep-analysis.workflow.ts
├── light-analysis.workflow.ts
├── xray-analysis.workflow.ts
├── bulk-analysis.workflow.ts
└── steps/
    ├── check-duplicate.step.ts
    ├── check-cache.step.ts
    ├── scrape-profile.step.ts      # Retry: 3x, exponential backoff
    ├── run-ai-analysis.step.ts     # Promise.all for parallel
    ├── cache-profile.step.ts
    ├── upsert-lead.step.ts
    └── save-results.step.ts

core/queues/
├── stripe-webhook.consumer.ts      # Process webhook events
└── analysis.consumer.ts            # Trigger workflows

core/durable-objects/
└── analysis-progress.do.ts         # Real-time progress + cancel
```

**Workflow Pattern:**
```typescript
export class DeepAnalysisWorkflow extends WorkflowEntrypoint {
  async run(event, step) {
    const progressTracker = getProgressTracker(run_id);
    
    // Step 1: Check duplicate
    await updateProgress('queued', 5, 'Checking duplicates');
    const duplicate = await step.do('check_duplicate', ...);
    if (duplicate) throw new Error('Already in progress');
    
    // Step 2: Check cache
    await updateProgress('scraping', 10, 'Checking cache');
    const cached = await step.do('check_cache', ...);
    
    // Step 3: Scrape (if cache miss)
    let profile = cached?.profile;
    if (!cached) {
      await updateProgress('scraping', 25, 'Scraping profile');
      profile = await step.do('scrape_profile', ...); // With retries
      await step.do('cache_profile', ...);
    }
    
    // Step 4: AI analysis (parallel)
    await updateProgress('analyzing', 50, 'Running AI (3 parallel calls)');
    const [core, outreach, personality] = await step.do('ai_analysis', async () =>
      Promise.all([
        aiService.executeCoreStrategy(profile, business),
        aiService.generateOutreach(profile, business),
        aiService.analyzePersonality(profile)
      ])
    );
    
    // Step 5: Upsert lead
    await updateProgress('saving', 80, 'Saving lead');
    const leadResult = await step.do('upsert_lead', ...);
    
    // Step 6: Save analysis
    await updateProgress('saving', 90, 'Saving results');
    await step.do('save_analysis', ...);
    
    // Complete
    await updateProgress('complete', 100, 'Done');
    
    // On error: Refund credits via deduct_credits(-amount)
  }
}
```

**Durable Object:**
- GET /progress → Client polls every 2s
- POST /update → Workflow updates
- POST /cancel → User cancels
- Cleanup alarm: 1hr after complete

**Success:** Cancel works, progress updates real-time, workflows complete

---

### Phase 9: Observability (Week 8)
**Build:**
```
infrastructure/monitoring/
├── analytics-engine.client.ts      # writeDataPoint for all requests
├── cost-tracker.service.ts         # Track Apify + AI costs
├── performance-tracker.service.ts  # Track P50/P95/P99
└── sentry.client.ts                # Error tracking
```

**Track:**
- Every analysis: type, duration, cost, cache_hit, credits_used
- Every AI call: model, tokens_in, tokens_out, cost, duration
- Every Apify call: duration, cost
- Every error: type, message, stack, context

**Query Examples:**
- P95 latency by analysis type
- Daily cost breakdown (Apify vs OpenAI)
- Cache hit rate over time
- Top cost drivers

**Success:** Can query metrics, identify bottlenecks, track costs

---

### Phase 10: Cron Jobs (Week 9)
**Build:**
```
core/cron/
├── monthly-renewal.job.ts          # 1st of month, 3 AM UTC
├── daily-cleanup.job.ts            # 2 AM UTC daily
├── failed-analysis-cleanup.job.ts  # Hourly
└── subscription-sync.job.ts        # Optional
```

**Monthly Renewal:**
1. Call `get_renewable_subscriptions()` RPC
2. For each subscription:
   - Check credit_ledger for renewal in past 23 hours (idempotency)
   - If not renewed: Grant credits via `deduct_credits(+amount)`
   - Update subscription period

**Daily Cleanup:**
1. Delete ai_usage_logs >90 days old
2. Hard-delete soft-deleted records >30 days old (analyses → leads → business_profiles → accounts)

**Failed Analysis Cleanup:**
1. Find analyses in 'pending'/'processing' status >5 minutes old
2. Mark as 'failed'
3. Refund credits via `deduct_credits(-amount)`

**wrangler.toml:**
```toml
[triggers.crons]
crons = [
  "0 3 1 * *",   # Monthly renewal (1st, 3 AM UTC)
  "0 2 * * *",   # Daily cleanup (2 AM UTC)
  "0 * * * *"    # Failed cleanup (hourly)
]
```

**Success:** Crons run on schedule, idempotency works, metrics logged

---

### Phase 11: Testing (Week 10-11)
**Build:**
```
tests/
├── unit/
│   ├── domain/         # 100% coverage
│   ├── application/    # 90% coverage
│   └── infrastructure/ # 70% coverage
├── integration/
│   ├── analysis-flow.test.ts
│   ├── credit-deduction.test.ts
│   └── webhook-idempotency.test.ts
└── e2e/
    ├── light-analysis.test.ts
    ├── deep-analysis.test.ts
    └── bulk-analysis.test.ts
```

**Critical Tests:**
- Duplicate analysis check prevents double charge
- Credit deduction atomic (no race conditions)
- Lead upsert works (update vs insert)
- Webhook idempotency (duplicate event ignored)
- Cron idempotency (renewal runs twice = same result)
- Cache hit/miss logic
- Workflow retries on failure
- Cancel analysis works
- Progress updates work

**Load Testing:**
- 100 concurrent analyses
- Verify no timeouts
- Verify no duplicate charges
- Verify performance targets met

**Success:** All tests pass, load test succeeds, ready for production

---

## 🚀 DEPLOYMENT CHECKLIST

### Pre-Deployment:
- [ ] All phases complete
- [ ] All tests passing
- [ ] Staging deployment successful
- [ ] Load testing completed (100 concurrent)
- [ ] Security audit passed
- [ ] Database migrations applied:
  - [ ] user_id → account_id migration
  - [ ] RLS fix (wrap auth.uid() in subquery)
  - [ ] Indexes: analyses.status, leads.account_username, etc.
- [ ] AWS Secrets populated
- [ ] Cloudflare bindings configured (KV, R2, Workflows, Queues, DO, Cron)

### Deployment:
- [ ] Deploy: `npm run deploy`
- [ ] Health check: `curl https://api.oslira.com/health`
- [ ] Test analysis with test account
- [ ] Verify crons scheduled
- [ ] Analytics Engine receiving data
- [ ] Sentry receiving errors

### Post-Deployment (24hr monitor):
- [ ] Performance targets met (LIGHT <11s, DEEP <25s, XRAY <25s)
- [ ] Cache hit rate increasing (target >30% after 1 week)
- [ ] Cost tracking accurate
- [ ] No duplicate analyses
- [ ] Credit deductions atomic
- [ ] Cancel works
- [ ] Crons running

---

## 📊 SUCCESS METRICS

### Performance Targets:
| Metric | Current | Target (Cold) | Target (Cached) |
|--------|---------|---------------|-----------------|
| LIGHT | 10-13s | 8-11s | 2-3s |
| DEEP | 30s | 18-23s | 12-15s |
| XRAY | 35s | 16-18s | 10-12s |
| Bulk 100 | 16min | 60-120s | N/A |
| Credit check | 500ms | <50ms | N/A |
| RLS queries | 450ms | <10ms | N/A |

### Cost Targets:
| Item | Current | Target | Savings |
|------|---------|--------|---------|
| AI costs | $300/mo | $210/mo | $90/mo |
| Worker CPU | $100/mo | $2/mo | $98/mo |
| **Total** | **$400/mo** | **$212/mo** | **$188/mo** |

### Reliability Targets:
| Metric | Current | Target |
|--------|---------|--------|
| Workflow success rate | N/A | >99% |
| Webhook loss | Possible | 0% (Queue retries) |
| Analysis failures | Lost forever | Auto-retry + refund |
| Duplicate analyses | Possible | 0% (5min check) |
| Credit race conditions | Possible | 0% (Atomic RPC) |

---

## 🔐 SECURITY PRINCIPLES

1. **RLS First:** Always use RLS client for user requests
2. **Admin Sparingly:** Service role only for: cron, webhooks, system ops
3. **Validate Input:** All user input validated before processing
4. **Rate Limit:** Prevent abuse (100 analyses/hour per account)
5. **Idempotent:** All operations safe to retry (webhooks, crons)
6. **Audit Trail:** Log all credit transactions with context
7. **Account-Based:** Credits belong to accounts, not users
8. **Subquery auth.uid():** Always wrap in (SELECT auth.uid()) for performance + correctness

---

## 📈 ARCHITECTURE PRINCIPLES

1. **Feature-First:** Each feature = domain/application/controllers/routes
2. **Ports & Adapters:** Domain defines interfaces, infrastructure implements
3. **Dependency Rule:** Outer → Inner, never reverse
4. **Single Responsibility:** Each module does ONE thing
5. **Fail-Safe Defaults:** Default deny, explicit grant
6. **Observable:** Every request tracked, every error classified
7. **Idempotent:** Safe to retry any operation
8. **Async by Default:** Use Workflows for any operation >5s

---

## ⚠️ KNOWN GAPS (Complete Before Starting)

**5 Missing Implementations:**

1. **Prompt Caching Implementation** (`infrastructure/ai/prompt-builder.service.ts`)
   - buildBusinessContext(business) → 800 tokens, CACHED
   - buildProfileData(profile) → Dynamic, NOT cached
   - buildMessagesWithCaching() → Structure for OpenAI/Claude

2. **Analysis Use Cases** (`features/analysis/application/use-cases/analyze-lead.usecase.ts`)
   - Orchestrates steps 1-5 of 12-step flow
   - Triggers workflow for steps 6-12

3. **Apify Adapter** (`infrastructure/scraping/apify.adapter.ts`)
   - Call Apify actor
   - Parse response
   - Transform to ProfileData

4. **AI Analysis Service** (`features/analysis/domain/services/ai-analysis.service.ts`)
   - Prompts for each analysis type
   - Call AI via Gateway
   - Parse JSON responses

5. **Queue Consumers** (`core/queues/stripe-webhook.consumer.ts`, `analysis.consumer.ts`)
   - Process webhook events (subscription.created, invoice.paid)
   - Trigger workflows

**Recommendation:** Complete these 5 before starting (~400 lines total)

---

## 📝 FINAL NOTES

**Implementation Approach:** Single-pass (not incremental)
1. Build all phases in new repo
2. Copy working code from old worker
3. Refactor into feature-first structure
4. Test in staging
5. Deploy all at once
6. Monitor 24hrs

**Why Single-Pass:** Team size supports it, reduces migration complexity

**Total Implementation Time:** 10-11 weeks (single developer, full-time)

**Rating:** Current 3/10 → Target 97/100 (final 3 points from production battle-testing)

---

**END OF SPECIFICATION v6.0**

**Context Preserved:** 100% | **Ready For:** Implementation after gap completion
