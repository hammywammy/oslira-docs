# 🏗️ OSLIRA CLOUDFLARE WORKER - IMPLEMENTATION PLAN v5.0

**Status:** Production Architecture - Ready for Single-Pass Implementation  
**Version:** 5.0 - Enterprise Grade  
**Approach:** Build complete system in one iteration  
**Last Updated:** 2025-01-20

---

## 📊 CURRENT STATE vs TARGET STATE

### **Current Worker: 3/10**

**Critical Failures:**
- ❌ Synchronous blocking (30s Worker timeout = analysis death)
- ❌ Service role everywhere (RLS bypassed = data leak risk)
- ❌ Zero observability (no cost tracking, no performance metrics)
- ❌ No progress tracking (users refresh = lost state)
- ❌ No retry logic (Apify fails = user loses credits forever)
- ❌ Mixed architecture (40+ scattered controller files)
- ❌ Race conditions (multiple deduct_credits = overdraft possible)
- ❌ user_id instead of account_id (wrong multi-tenant model)

**What Actually Works:**
- ✅ AWS Secrets integration
- ✅ Path aliases configured
- ✅ Domain folders exist (just poorly organized)

---

### **Target Worker: 97/100**

**Core Infrastructure:**
- ✅ Workflows = async orchestration (no timeout limits)
- ✅ Dual Supabase client (RLS enforced where appropriate)
- ✅ Cloudflare AI Gateway (30-40% cost savings, FREE)
- ✅ Durable Objects (real-time progress + cancel capability)
- ✅ Feature-first architecture (vertical slices)
- ✅ Comprehensive observability (Analytics Engine + Sentry)
- ✅ Cloudflare Cron Triggers (monthly renewals, daily cleanup)

**Performance Gains:**
- LIGHT: 10-13s → 6s avg (with 50% cache hit rate)
- DEEP: 30s → 18-23s (full AI parallelization)
- XRAY: 35s → 16-18s (optimized model selection)
- Bulk 100 profiles: 16min → 60-120s (batched processing)

**Cost Reductions:**
- AI costs: $300/mo → $210/mo (prompt caching + AI Gateway)
- Worker CPU: $100/mo → $2/mo (Workflows handle heavy lifting)
- Total savings: **$188/month**

**Why not 100/100?** Needs production battle-testing for final edge cases.

---

## 🎯 WHAT MUST BE BUILT

### **PHASE 0: Foundation**

**Infrastructure Setup:**
- Validate all Cloudflare bindings (KV, R2, Workflows, Queues, Durable Objects)
- Apply RLS performance fix to Supabase (wrap `auth.uid()` in subqueries = 100x faster)
- Health check endpoints showing binding status
- Verify SECURITY DEFINER on all Supabase RPC functions

**Configuration:**
- AWS Secrets caching (5-minute TTL per secret)
- Environment detection (production/staging)
- Path alias resolution via Wrangler esbuild

---

### **PHASE 1: Feature-First Architecture**

**Vertical Slices (Build in Order):**

```
features/
├── credits/
│   ├── domain/              # Credit business rules, value objects
│   ├── application/         # Use cases (get balance, deduct credits)
│   ├── controllers/         # HTTP endpoints
│   └── routes.ts
│
├── auth/
│   ├── domain/              # JWT validation rules
│   ├── application/         # Verify token use case
│   ├── controllers/         # Auth endpoints
│   └── routes.ts
│
├── business/
│   ├── domain/              # Business profile entity, ICP logic
│   ├── application/         # CRUD use cases
│   ├── controllers/         # Business profile endpoints
│   └── routes.ts
│
└── analysis/
    ├── domain/              # Lead scoring, analysis rules
    ├── application/         # Analyze use cases (single, bulk, anonymous)
    ├── controllers/         # Analysis endpoints
    └── routes.ts
```

**Why This Order:**
1. Credits = simplest, validates pattern
2. Auth = foundation for everything
3. Business = user setup flow
4. Analysis = core product (needs 1-3 first)

---

### **PHASE 2: Infrastructure Adapters**

```
infrastructure/
├── database/
│   ├── supabase.client.ts          # Dual client factory
│   └── repositories/
│       ├── base.repository.ts      # Generic CRUD
│       ├── credits.repository.ts
│       ├── business.repository.ts
│       ├── leads.repository.ts     # ✅ Includes upsertLead()
│       └── analysis.repository.ts
│
├── ai/
│   ├── openai.adapter.ts
│   ├── claude.adapter.ts
│   └── ai-gateway.client.ts        # Routes through Cloudflare AI Gateway
│
├── scraping/
│   └── apify.adapter.ts
│
├── cache/
│   └── r2-cache.service.ts         # Global profile caching
│
└── monitoring/
    ├── analytics-engine.client.ts  # FREE observability
    ├── sentry.client.ts            # Critical errors only
    ├── cost-tracker.service.ts     # Apify + AI costs
    └── performance-tracker.service.ts
```

**Key Implementations:**

**Dual Supabase Client:**
- User client (anon key) = RLS enforced for reads
- Admin client (service role) = System operations only

**Lead Upsert:**
- Check existing by: username + account_id + business_profile_id + deleted_at IS NULL
- If exists → Update profile data, return { leadId, isNew: false }
- If not → Insert new, return { leadId, isNew: true }

**R2 Global Cache:**
- TTL: 24h (light), 12h (deep), 6h (xray)
- 50%+ cache hit rate after Month 1
- Cache hits: <1s vs 8s scraping

---

### **PHASE 3: Shared Layer**

```
shared/
├── middleware/
│   ├── auth.middleware.ts          # Extract user from JWT
│   ├── rate-limit.middleware.ts    # KV-based
│   ├── error.middleware.ts         # Global error handler
│   └── analytics.middleware.ts     # Track every request
│
├── utils/
│   ├── logger.util.ts
│   ├── validation.util.ts
│   └── response.util.ts
│
└── types/
    └── (feature-specific types live in features/)
```

---

### **PHASE 4: Async Orchestration**

```
core/workflows/
├── deep-analysis.workflow.ts       # No timeout limits
├── bulk-analysis.workflow.ts
└── steps/
    ├── check-duplicate.step.ts     # ✅ Prevent duplicate analyses
    ├── scrape-profile.step.ts      # Retries on Apify failures
    ├── run-ai-analysis.step.ts     # Full parallelization
    └── save-results.step.ts

core/queues/
├── stripe-webhook.consumer.ts      # ✅ Idempotency check before processing
└── analysis.consumer.ts

core/durable-objects/
└── analysis-progress.do.ts         # Real-time progress + cancel
```

**Workflow Pattern:**
```
1. Check cache (R2)
2. Check duplicate analysis in progress
3. Deduct credits (atomic RPC)
4. Scrape profile (with retry)
5. Run AI analysis (parallel calls)
6. Cache result (R2)
7. Save to database
8. Update progress (Durable Object)
```

**Durable Object Specs:**
- Scope: One DO per run_id
- State: { run_id, status, progress, current_step, should_cancel }
- Persistence: Ephemeral (cleanup 1 hour after completion)
- Transport: HTTP polling every 2s (not WebSocket)
- Fallback: If DO down → return basic status from database

---

### **PHASE 5: Cron Jobs**

```
core/cron/
├── monthly-renewal.job.ts          # 1st of month, 3 AM UTC
├── daily-cleanup.job.ts            # Daily, 2 AM UTC
├── failed-analysis-cleanup.job.ts  # Hourly
├── subscription-sync.job.ts        # Every 6 hours (optional)
└── invoice-reconciliation.job.ts   # Daily, 4 AM UTC (optional)
```

**Must-Have Cron Jobs:**

**1. Monthly Credit Renewal**
- Query `get_renewable_subscriptions()` RPC
- Call `deduct_credits()` with positive amount (grants credits)
- Update subscription period (advance 1 month)
- Track: total processed, success/failure counts

**2. Daily Cleanup**
- Delete `ai_usage_logs` older than 90 days
- Hard-delete soft-deleted records older than 30 days (leads, analyses, business_profiles, accounts)
- Track: rows deleted per table

**3. Failed Analysis Cleanup**
- Find analyses stuck in pending/processing >15 minutes
- Mark as failed
- Refund credits via `deduct_credits()`

**Cron Configuration:**
```toml
# wrangler.toml
[triggers]
crons = [
  "0 3 1 * *",   # Monthly renewal
  "0 2 * * *",   # Daily cleanup
  "0 * * * *"    # Hourly failed analysis cleanup
]
```

---

### **PHASE 6: Critical Fixes**

**1. Webhook Idempotency**
```
Stripe webhook arrives
  ↓
Validate signature
  ↓
✅ CHECK webhook_events table for stripe_event_id
  ↓ (if exists)
Return 200 immediately
  ↓ (if not exists)
INSERT webhook_events (ON CONFLICT DO NOTHING)
  ↓
Queue for processing
  ↓
Return 200 to Stripe
```

**2. account_id Not user_id**
- All credit operations use account_id
- All queries filter by account_id
- Credits belong to accounts, not users
- Users can belong to multiple accounts

**3. RLS Performance**
Apply to ALL Supabase policies:
```sql
-- ❌ SLOW
WHERE user_id = auth.uid()

-- ✅ FAST (100x)
WHERE user_id = (SELECT auth.uid())
```

---

## 🚀 OPTIMIZATIONS INCLUDED

### **Performance Optimizations**

**1. R2 Global Profile Caching**
- 50%+ cache hit rate after Month 1
- Cache hits: <1s vs 8s scraping
- Different TTLs per analysis type

**2. Prompt Caching (Business Context)**
- Business context (800 tokens): 90% cost reduction after first use
- Speed: 2-4s faster per analysis
- Works for OpenAI (auto) and Claude (cache_control)

**3. Full AI Parallelization**
```javascript
// DEEP: Sequential (OLD)
core → outreach → personality = 19s

// DEEP: Parallel (NEW)
Promise.all([core, outreach, personality]) = 15s (saved 4s)

// XRAY: Parallel (NEW)
Promise.all([psycho, commercial, outreach, personality]) = 10s (saved 12s)
```

**4. Intelligent Model Selection**
- LIGHT: gpt-4o-mini (fast, cheap, good at JSON)
- DEEP core: gpt-5-mini (balanced)
- XRAY psychographic: gpt-5 (premium for deep insight)
- All others: gpt-5-mini
- Reasoning effort optimized per task

**5. Async Execution**
```
User clicks "Analyze"
  ↓ (400ms)
Response: { run_id, status: 'queued', poll_url }
  ↓
Frontend polls /runs/:run_id every 2s
  ↓
Shows real-time progress via Durable Object
  ↓
On complete: Stop polling, show results
```

**6. Bulk Analysis Batching**
- Respects Apify concurrency limit (10 concurrent)
- Splits 100 profiles into 10 batches
- Processes sequentially: 60-120s total
- Progress tracking per batch

---

### **Cost Optimizations**

**Comprehensive Cost Tracking:**
```javascript
costTracker.trackApifyCall(postsScraped, duration)
costTracker.trackAICall(model, tokensIn, tokensOut, duration)
costTracker.getBreakdown() // → { apify, ai, total, margins }
```

**AI Gateway Integration:**
- One-line baseURL change
- 30-40% cost reduction (caches duplicate prompts)
- Automatic fallback (OpenAI down → Claude)
- Free analytics (track costs per user)

---

### **Observability**

**Analytics Engine (FREE):**
```javascript
env.ANALYTICS_ENGINE.writeDataPoint({
  blobs: [run_id, username, analysisType],
  doubles: [cost, duration_ms, score],
  indexes: [userId, accountId]
})
```

**Queries:**
- Which users drive 80% of AI costs?
- P95 latency per endpoint
- Cache hit rate over time
- Analysis success rate

**Sentry (PAID - Critical Only):**
- Only log CRITICAL errors
- Not INFO/WARN (those go to Analytics Engine)

**Performance Tracking:**
```javascript
perf.startStep('cache_check')
perf.endStep('cache_check')
perf.getBreakdown() // → identifies bottlenecks
```

---

## ❌ WILL NOT IMPLEMENT

**Rejected Features:**
1. **Gradual Deployments** - Have staging, all-at-once fine
2. **D1 for Caching** - R2 + KV sufficient
3. **Service Bindings** - Monolith works, clear domain separation
4. **Workers AI** - OpenAI/Claude already contracted
5. **Browser Rendering API** - Not in roadmap
6. **AutoRAG/Vectorize** - "Similar influencers" not in MVP
7. **Hyperdrive** - Supabase already fast, not bottleneck
8. **Supabase DB Webhooks** - Workflows notify directly

---

## 🔮 DEFERRED TO FUTURE

**Post-MVP Enhancements:**

**Apify Fallback (If needed):**
- Trigger: Apify downtime >1 hour/month OR costs >30% budget
- Add: Bright Data adapter, ScraperAPI adapter
- Automatic failover on errors

**Advanced Caching (Month 2+):**
- Semantic caching for XRAY (embeddings-based)
- Batch API for multiple AI calls (50% discount)

**R2 Cache Expiration Cron:**
- Daily cleanup of expired cached profiles
- Storage cost optimization

---

## 🎯 CRITICAL DECISIONS LOCKED

### **1. Dual Supabase Client Pattern**
**MANDATORY for security:**
```typescript
// User client (RLS enforced)
createUserClient(userId): SupabaseClient

// Admin client (bypass RLS for system ops)
createAdminClient(): SupabaseClient
```

### **2. AI Gateway Integration**
**MANDATORY (FREE, saves money):**
```typescript
// Before
baseURL: "https://api.openai.com/v1"

// After
baseURL: "https://gateway.ai.cloudflare.com/v1/{account}/{gateway}/openai"
```

### **3. Workflow Error Handling Philosophy**
**Retry infrastructure failures, NOT business logic:**
```typescript
// ✅ RETRY: Apify timeout, OpenAI 503
// ❌ NO RETRY: Insufficient credits, invalid input
```

### **4. Secret Rotation**
**Accept 5-minute cache delay** - Secrets rarely rotate, simplicity wins

### **5. CI/CD**
**Manual deployment only:**
```bash
npm test        # Vitest suite
npm run typecheck
npm run deploy  # Manual control
```

### **6. Cron Jobs**
**Cloudflare Cron Triggers, NOT Supabase pg_cron:**
- Already part of Worker architecture
- Better observability (Analytics Engine + Sentry)
- Easy testing via manual trigger endpoints
- Can call external APIs (future-proof)

---

## 🏗️ FOLDER STRUCTURE

```
src/
├── features/              # Vertical slices
│   ├── credits/
│   ├── auth/
│   ├── business/
│   └── analysis/
│
├── core/                  # Horizontal services
│   ├── workflows/
│   ├── queues/
│   ├── durable-objects/
│   └── cron/
│
├── infrastructure/        # External adapters
│   ├── database/
│   ├── ai/
│   ├── scraping/
│   ├── cache/
│   └── monitoring/
│
└── shared/               # Pure utilities
    ├── middleware/
    ├── utils/
    └── types/
```

---

## 🔧 TECHNOLOGY STACK

| Layer | Technology | Why |
|-------|-----------|-----|
| Runtime | Cloudflare Workers | Serverless, global edge |
| Framework | Hono v4 | Fast, TypeScript-first |
| Database | Supabase (Postgres) | RLS, real-time, managed |
| Secrets | AWS Secrets Manager | Centralized, rotatable |
| AI Gateway | Cloudflare AI Gateway | FREE caching + fallback |
| Workflows | Cloudflare Workflows | No timeout limits |
| Queues | Cloudflare Queues | Fire-and-forget retries |
| Progress | Durable Objects | Real-time state + cancel |
| Caching | R2 + KV | Global profile cache |
| Observability | Analytics Engine (FREE) | Track everything |
| Error Tracking | Sentry | Critical only |
| Testing | Vitest + Workers pool | Runs in Workers runtime |

---

## 📈 SUCCESS METRICS

### **Performance**
| Metric | Current | Target | Improvement |
|--------|---------|--------|-------------|
| LIGHT cold | 10-13s | 8-11s | 20% faster |
| LIGHT cached | N/A | 2-3s | 90% faster |
| DEEP cold | 30s | 18-23s | 30% faster |
| XRAY cold | 35s | 16-18s | 50% faster |
| Bulk 100 | 16min | 60-120s | 87% faster |
| Credit check | 500ms | <50ms | 90% faster |

### **Cost**
| Item | Current | Target | Savings |
|------|---------|--------|---------|
| AI costs | $300/mo | $210/mo | $90/mo |
| Worker CPU | $100/mo | $2/mo | $98/mo |
| **Total** | **$400/mo** | **$212/mo** | **$188/mo** |

### **Reliability**
| Metric | Current | Target |
|--------|---------|--------|
| Workflow success | N/A | >99% |
| Webhook loss | Possible | 0% (Queue retries) |
| Analysis failures | Lost forever | Auto-retry + refund |

---

## 🎯 ARCHITECTURE PRINCIPLES

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

### **Infrastructure**
- [ ] Validate Cloudflare bindings (KV, R2, Workflows, Queues, DO)
- [ ] Apply RLS fix to Supabase (wrap auth.uid())
- [ ] Add cron triggers to wrangler.toml
- [ ] Configure Analytics Engine binding
- [ ] Set up Sentry project

### **Database**
- [ ] Verify get_renewable_subscriptions() exists
- [ ] Verify deduct_credits() exists
- [ ] Add indexes (analyses.status, ai_usage_logs.created_at)
- [ ] Verify SECURITY DEFINER on all RPC functions
- [ ] Test cascading deletes

### **Features**
- [ ] Build credits feature (validates pattern)
- [ ] Build auth feature
- [ ] Build business profiles feature
- [ ] Build analysis feature (core product)

### **Infrastructure Adapters**
- [ ] Dual Supabase client factory
- [ ] Lead upsert repository method
- [ ] R2 cache service
- [ ] AI Gateway integration
- [ ] Cost tracker
- [ ] Performance tracker

### **Async Infrastructure**
- [ ] Deep analysis workflow (still need light and xray)
- [ ] Bulk analysis workflow
- [ ] Stripe webhook queue consumer
- [ ] Analysis progress Durable Object

### **Cron Jobs**
- [ ] Monthly credit renewal
- [ ] Daily cleanup
- [ ] Failed analysis cleanup
- [ ] Manual trigger admin endpoints

### **Critical Fixes**
- [ ] Webhook idempotency check
- [ ] Change user_id to account_id everywhere
- [ ] Duplicate analysis check before deduction

### **Testing**
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

### **Current Worker: 3/10**
- Blocking synchronous operations
- Security holes (RLS bypassed)
- Zero observability
- Poor architecture
- Race conditions
- Wrong multi-tenancy model

### **Target Worker: 97/100**
- Async orchestration (no timeouts)
- Defense-in-depth security
- Full observability
- Clean feature-first architecture
- Comprehensive cost tracking
- Production-ready with all optimizations

### **Why not 100/100?**
Final 3 points earned through production battle-testing and real-world edge case handling.

---

**END OF IMPLEMENTATION PLAN**

All decisions finalized. All technologies selected. Ready to build in single pass.

## Implementation detials: 
