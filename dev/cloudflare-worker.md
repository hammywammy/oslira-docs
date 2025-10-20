## first i need to determine what things i need to setup and what keys to put into aws 
## then i need to think about the acutaly exectution of my analysis and find a true optimized plan with my current system

# 🏗️ OSLIRA CLOUDFLARE WORKER - IMPLEMENTATION PLAN

**Status:** Production Architecture Finalized  
**Version:** 4.0 - Enterprise Grade  
**Timeline:** 10 weeks to production-ready foundation  
**Last Updated:** 2025-01-20

---

## 📊 CURRENT vs TARGET STATE

### **Current Worker: 4/10**

**What's Wrong:**
- ❌ Everything synchronous (30s timeouts, no retry on failure)
- ❌ Service role bypasses RLS everywhere (security hole)
- ❌ No observability (can't track AI costs per user)
- ❌ No real-time progress (users refresh and lose status)
- ❌ Mixed concerns (40+ controller files, hard to navigate)
- ❌ No circuit breakers (external API failures cascade)

**What's Good:**
- ✅ AWS Secrets integration works
- ✅ Path aliases setup correct
- ✅ Domain separation exists (api/domain/infrastructure/shared)

---

### **Target Worker: 95/100**

**What We're Building:**
- ✅ Workflows = async orchestration (no 30s timeout)
- ✅ Dual Supabase client (RLS enforced for reads, service role for system ops)
- ✅ AI Gateway = cost savings + fallback + observability
- ✅ Durable Objects = real-time WebSocket progress
- ✅ Feature-first architecture (vertical slices, easy to find code)
- ✅ Comprehensive testing (Vitest in Workers runtime)
- ✅ Production observability (Analytics Engine + targeted Sentry)

**Why not 100/10?** Nothing is perfect until battle-tested in production.

---

## ✅ FINALIZED - WILL IMPLEMENT

### **Phase 0: Foundation (Week 1)**
- ✅ AWS Secrets Manager integration (DONE)
- ✅ Config manager with caching (DONE)
- ✅ Environment detection (production/staging) (DONE)
- 🆕 Cloudflare bindings validation (KV, R2, Workflows, Queues, DO)
- 🆕 Health check endpoints with binding status
- 🆕 Apply RLS performance fix (wrap auth.uid() in subqueries)

### **Phase 1-2: Feature-First Architecture (Weeks 2-6)**

**Build vertical slices in order:**

#### **Credits Feature (Week 2)** - Simplest, validates pattern
```
features/credits/
├── domain/               # Credit business rules, calculator
├── application/          # Deduct/grant use cases
├── controllers/          # Balance/transactions endpoints
└── routes.ts
```

#### **Auth Feature (Week 3)** - Foundation for everything
```
features/auth/
├── domain/               # JWT validation rules
├── application/          # Login/verify use cases
├── controllers/          # Auth endpoints
└── routes.ts
```

#### **Business Profiles Feature (Week 4)** - User setup
```
features/business/
├── domain/               # Business rules, ICP logic
├── application/          # CRUD use cases
├── controllers/          # Business endpoints
└── routes.ts
```

#### **Analysis Feature (Weeks 5-6)** - Core product
```
features/analysis/
├── domain/               # Lead scoring, analysis rules
├── application/          # Analyze use cases
├── controllers/          # Analysis endpoints
└── routes.ts
```

### **Phase 3: Infrastructure Adapters (Ongoing)**

Built alongside features as needed:

```
infrastructure/
├── database/
│   ├── supabase.client.ts          # Dual client factory
│   ├── repositories/
│   │   ├── base.repository.ts      # Generic CRUD
│   │   ├── user.repository.ts
│   │   ├── credits.repository.ts
│   │   ├── business.repository.ts
│   │   ├── leads.repository.ts
│   │   └── analysis.repository.ts
├── ai/
│   ├── openai.adapter.ts
│   ├── claude.adapter.ts
│   └── ai-gateway.client.ts        # FREE cost savings
└── scraping/
    └── apify.adapter.ts
```

### **Phase 4: Shared Layer (Weeks 2-3)**

```
shared/
├── middleware/
│   ├── auth.middleware.ts          # JWT extraction
│   ├── rate-limit.middleware.ts    # KV-based
│   ├── error.middleware.ts         # Global handler
│   └── analytics.middleware.ts     # Track every request
├── utils/
│   ├── logger.util.ts
│   ├── validation.util.ts
│   └── response.util.ts
└── types/
    └── (feature-specific types stay in features/)
```

### **Phase 5: Async Orchestration (Weeks 7-8)**

```
core/workflows/
├── deep-analysis.workflow.ts       # 20min jobs, no timeout
├── bulk-analysis.workflow.ts
└── steps/
    ├── scrape-profile.step.ts      # Each step retries independently
    ├── run-ai-analysis.step.ts
    └── save-results.step.ts

core/queues/
├── stripe-webhook.consumer.ts      # Fire-and-forget retries
└── email.consumer.ts

core/durable-objects/
└── analysis-progress.do.ts         # WebSocket real-time updates
```

### **Phase 6: Observability (Week 9)**

```
infrastructure/monitoring/
├── analytics-engine.client.ts      # FREE - log everything
├── sentry.client.ts                # PAID - critical errors only
├── error-classifier.ts             # INFO/WARN/ERROR/CRITICAL
└── retry-strategies.ts             # Critical/Standard/FastFail

shared/middleware/
└── analytics.middleware.ts         # Dimensions: user, cost, latency
```

### **Phase 7: Testing Infrastructure (Week 10)**

```
tests/
├── unit/
│   ├── features/
│   │   ├── credits/domain/*.test.ts
│   │   ├── auth/application/*.test.ts
│   │   └── analysis/domain/*.test.ts
│   └── infrastructure/
│       └── database/*.test.ts
├── integration/
│   ├── credits/balance-endpoint.test.ts
│   └── analysis/analyze-endpoint.test.ts
└── e2e/
    └── analysis-workflow.test.ts
```

**Testing Strategy:**
- ✅ Vitest with `@cloudflare/vitest-pool-workers` (runs IN Workers runtime)
- ✅ Domain layer: 100% coverage (pure business logic)
- ✅ Application layer: 90% coverage (use cases)
- ✅ Infrastructure: 70% coverage (mocked externals)
- ✅ Controllers: 80% coverage (API endpoints)

---

## ❌ FINALIZED - WILL NOT IMPLEMENT

### **1. Gradual Deployments**
**Why:** Have staging environment. All-at-once deployment fine for team size. Adds complexity for minimal benefit.

### **2. D1 for Caching Layer**
**Why:** Supabase is source of truth. R2 + KV handle caching. Adding D1 = unnecessary layer.

### **3. Service Bindings for Microservices**
**Why:** Monolith works fine. Domain separation clear via features/. Only split if CPU limits hit (unlikely).

### **4. Workers AI (On-Platform Models)**
**Why:** OpenAI/Claude already contracted. Switching = quality risk for minimal savings.

### **5. Browser Rendering API**
**Why:** Not in feature roadmap. Add only when users request "export PDF".

### **6. AutoRAG / Vectorize**
**Why:** "Similar influencers" feature not in MVP. Add when building that capability.

### **7. Hyperdrive for Supabase**
**Why:** Supabase already sub-10ms. Not bottleneck. Test P95 latency first.

### **8. Supabase Database Webhooks**
**Why:** Workflows notify directly. No need for DB → Webhook → Worker callback.

---

## 🔮 DEFERRED - FUTURE ROADMAP

### **Phase 8: Apify Fallback (Post-MVP)**
**Status:** Marked for future implementation  
**Trigger:** If Apify downtime exceeds 1 hour/month OR costs >30% of budget

```
infrastructure/scraping/
├── apify.adapter.ts
├── bright-data.adapter.ts          # Fallback option 1
└── scraper-api.adapter.ts          # Fallback option 2
```

**Implementation:**
- AI Gateway-style routing
- Automatic failover on errors
- Cost tracking per provider

---

## 📋 IMPLEMENTATION PHASES

### **Week 1: Phase 0 - Foundation**

**Deliverables:**
- ✅ Validate all Cloudflare bindings (KV, R2, Workflows, Queues, DO)
- ✅ Apply RLS performance fix (wrap all `auth.uid()` calls)
- ✅ Health check endpoints
- ✅ Verify SECURITY DEFINER on all RPC functions

**Success Criteria:**
- Worker boots without errors
- All bindings accessible
- Health check returns 200 with binding status
- Database queries 100x faster (RLS optimization)

---

### **Week 2: Phase 1 - Credits Feature**

**Deliverables:**
```
features/credits/
├── domain/
│   ├── credit-amount.vo.ts         # Value object (prevents negative)
│   └── credit-calculator.service.ts # Business rules
├── application/
│   ├── get-balance.usecase.ts
│   ├── get-transactions.usecase.ts
│   └── ports/
│       └── credit.repository.interface.ts
├── controllers/
│   ├── balance.controller.ts       # GET /credits/balance
│   └── transactions.controller.ts  # GET /credits/transactions
└── routes.ts

infrastructure/database/repositories/
└── credits.repository.ts            # Implements interface

tests/unit/features/credits/
├── domain/credit-calculator.test.ts
└── application/get-balance.test.ts
```

**Success Criteria:**
- ✅ Pattern validated (domain → application → infrastructure)
- ✅ Endpoints return correct data
- ✅ Tests pass (domain + application)
- ✅ RLS enforced (user sees only their balance)

---

### **Week 3: Phase 2 - Auth Feature**

**Deliverables:**
```
features/auth/
├── domain/
│   └── jwt-validator.service.ts
├── application/
│   └── verify-token.usecase.ts
├── controllers/
│   └── auth.controller.ts
└── routes.ts

shared/middleware/
└── auth.middleware.ts              # Extract user from JWT
```

**Success Criteria:**
- ✅ JWT validation works
- ✅ Middleware extracts user ID
- ✅ Unauthorized requests blocked

---

### **Week 4: Phase 3 - Business Profiles Feature**

**Deliverables:**
```
features/business/
├── domain/
│   ├── business-profile.entity.ts
│   └── icp-matcher.service.ts
├── application/
│   ├── create-profile.usecase.ts
│   ├── update-profile.usecase.ts
│   └── ports/
│       └── business.repository.interface.ts
├── controllers/
│   └── profiles.controller.ts
└── routes.ts

infrastructure/database/repositories/
└── business.repository.ts

infrastructure/
└── database/
    └── supabase.client.ts          # ✅ Dual client factory
```

**Success Criteria:**
- ✅ Dual Supabase client pattern implemented
- ✅ User client (anon key) for reads
- ✅ Admin client (service role) for system operations
- ✅ CRUD operations work
- ✅ RLS enforced

---

### **Weeks 5-6: Phase 4 - Analysis Feature (Core Product)**

**Deliverables:**
```
features/analysis/
├── domain/
│   ├── entities/
│   │   ├── analysis.entity.ts
│   │   └── lead.entity.ts
│   ├── value-objects/
│   │   ├── analysis-score.vo.ts
│   │   └── instagram-handle.vo.ts
│   └── services/
│       └── lead-scorer.service.ts
├── application/
│   ├── use-cases/
│   │   ├── analyze-lead.usecase.ts
│   │   ├── bulk-analyze.usecase.ts
│   │   └── anonymous-analyze.usecase.ts
│   ├── ports/
│   │   ├── ai-provider.interface.ts
│   │   └── scraper.interface.ts
│   └── dtos/
│       └── analysis.dto.ts
├── controllers/
│   ├── analyze.controller.ts
│   ├── bulk-analyze.controller.ts
│   └── anonymous-analyze.controller.ts
└── routes.ts

infrastructure/
├── ai/
│   ├── openai.adapter.ts
│   ├── claude.adapter.ts
│   └── ai-gateway.client.ts        # ✅ Route through AI Gateway
└── scraping/
    └── apify.adapter.ts
```

**Success Criteria:**
- ✅ AI Gateway integrated (FREE caching + fallback)
- ✅ Analysis creates lead, deducts credits, runs AI
- ✅ Bulk analysis queues multiple jobs
- ✅ Anonymous analysis rate-limited (KV)
- ✅ All operations tested

---

### **Weeks 7-8: Phase 5 - Workflows + Durable Objects**

**Deliverables:**
```
core/workflows/
├── deep-analysis.workflow.ts
├── bulk-analysis.workflow.ts
└── steps/
    ├── scrape-profile.step.ts
    ├── run-ai-analysis.step.ts
    └── save-results.step.ts

core/queues/
└── stripe-webhook.consumer.ts

core/durable-objects/
└── analysis-progress.do.ts

wrangler.toml
# Add bindings:
[[workflows]]
binding = "ANALYSIS_WORKFLOW"
class_name = "DeepAnalysisWorkflow"

[durable_objects]
bindings = [
  { name = "ANALYSIS_PROGRESS", class_name = "AnalysisProgressTracker" }
]

[[queues.producers]]
binding = "STRIPE_WEBHOOKS_QUEUE"
queue = "stripe-webhooks"
```

**Success Criteria:**
- ✅ Workflows run 20min+ analyses without timeout
- ✅ Durable Objects provide WebSocket real-time progress
- ✅ Each workflow step retries independently
- ✅ User can refresh browser, progress persists
- ✅ Stripe webhooks retry via Queue if fail

---

### **Week 9: Phase 6 - Observability**

**Deliverables:**
```
infrastructure/monitoring/
├── analytics-engine.client.ts
├── sentry.client.ts
├── error-classifier.ts
└── retry-strategies.ts

shared/middleware/
└── analytics.middleware.ts
```

**Analytics Engine Integration:**
```typescript
// Every request tracked with dimensions:
{
  endpoint: '/v1/analyze',
  userId: 'user_123',
  accountId: 'acc_456',
  analysisType: 'deep',
  aiCost: 0.015,
  latencyMs: 450,
  statusCode: 200,
  creditsUsed: 2
}
```

**Success Criteria:**
- ✅ Every request logged to Analytics Engine
- ✅ Only CRITICAL errors go to Sentry
- ✅ Can query: "Which users drive 80% of AI costs?"
- ✅ Can query: "P95 latency per endpoint"
- ✅ Retry strategies applied (Critical/Standard/FastFail)

---

### **Week 10: Phase 7 - Testing Infrastructure**

**Deliverables:**
```
vitest.config.ts
tests/
├── unit/
├── integration/
└── e2e/

package.json
"scripts": {
  "test": "vitest",
  "test:unit": "vitest run tests/unit",
  "test:integration": "vitest run tests/integration",
  "test:e2e": "vitest run tests/e2e"
}
```

**Success Criteria:**
- ✅ Domain layer: 100% coverage
- ✅ Application layer: 90% coverage
- ✅ All tests run in Workers runtime (via vitest-pool-workers)
- ✅ Integration tests use SELF binding
- ✅ CI validates tests before deploy

---

## 🎯 CRITICAL DECISIONS LOCKED IN

### **1. Dual Supabase Client Pattern**
**Status:** MANDATORY

```typescript
// infrastructure/database/supabase.client.ts
export class SupabaseClientFactory {
  // For reads - RLS enforced
  createUserClient(userId: string): SupabaseClient {
    return createClient(url, anonKey);
  }
  
  // For system operations - bypass RLS
  createAdminClient(): SupabaseClient {
    return createClient(url, serviceRoleKey);
  }
}
```

**Why:** Security-in-depth. Even if auth bugs exist, RLS protects data.

---

### **2. AI Gateway Integration**
**Status:** MANDATORY (FREE, saves money)

```typescript
// Before
const openai = new OpenAI({
  baseURL: "https://api.openai.com/v1"
});

// After (one-line change)
const openai = new OpenAI({
  baseURL: "https://gateway.ai.cloudflare.com/v1/{account}/{gateway}/openai"
});
```

**Benefits:**
- 30-40% cost reduction (caching duplicate prompts)
- Automatic fallback (OpenAI down → Claude)
- Free analytics (track costs per user)
- Zero cost, zero downside

---

### **3. Workflow Error Handling**
**Status:** PHILOSOPHY LOCKED

**Only retry infrastructure failures, not business logic failures:**

```typescript
// ✅ RETRY: Infrastructure failures
const profile = await step.do('scrape', async () => {
  return await scrapeProfile(username); // Retries if Apify times out
});

// ❌ NO RETRY: Business logic failures
const credits = await step.do('check_credits', async () => {
  if (userCredits < 2) {
    throw new BusinessLogicError('Insufficient credits');
  }
});
// Workflow stops immediately, no retry

// ✅ RETRY: AI API failures
const analysis = await step.do('analyze', async () => {
  return await analyzeWithAI(profile); // Retries if OpenAI 503
});
```

---

### **4. Secret Rotation**
**Status:** Accept 5-minute cache delay

**Reasoning:** Secrets not rotating frequently. Simple > complex.

---

### **5. CI/CD**
**Status:** Manual deployment only

**Workflow:**
```bash
npm test                # Run Vitest suite
npm run typecheck       # Verify types
npm run deploy          # wrangler deploy (manual)
```

**No GitHub Actions.** You control deploys.

---

## 🔧 TECHNICAL SPECIFICATIONS

### **Folder Structure**

```
src/
├── features/           # Vertical slices
│   ├── credits/
│   ├── auth/
│   ├── business/
│   └── analysis/
├── core/              # Horizontal services
│   ├── ai/
│   ├── scraping/
│   ├── workflows/
│   ├── queues/
│   └── durable-objects/
├── infrastructure/    # External world
│   ├── config/
│   ├── database/
│   ├── cache/
│   └── monitoring/
└── shared/           # Pure utilities
    ├── middleware/
    ├── utils/
    └── types/
```

---

### **Key Technologies**

| Layer | Technology | Version |
|-------|-----------|---------|
| Runtime | Cloudflare Workers | Latest |
| Framework | Hono | ^4.0.0 |
| Database | Supabase (Postgres) | Latest |
| Secrets | AWS Secrets Manager | - |
| AI Gateway | Cloudflare AI Gateway | FREE |
| Workflows | Cloudflare Workflows | - |
| Queues | Cloudflare Queues | - |
| Real-time | Durable Objects | - |
| Testing | Vitest + @cloudflare/vitest-pool-workers | 2.0.x |
| Observability | Workers Analytics Engine (FREE) | - |
| Error Tracking | Sentry (critical only) | - |

---

### **Architecture Principles**

1. **Feature-first:** Each feature owns its domain/application/controllers
2. **Ports & Adapters:** Domain defines interfaces, infrastructure implements
3. **Dependency Rule:** Outer layers depend on inner, never reverse
4. **Single Responsibility:** Each module does ONE thing well
5. **Fail-safe Defaults:** Default deny, explicit grant
6. **Observable:** Every request tracked, every error classified

---

## 📊 SUCCESS METRICS

### **Performance**

| Metric | Current | Target |
|--------|---------|--------|
| P95 Analysis Latency | 30s (timeout) | 20s (async) |
| Credit Check Latency | 500ms | <50ms |
| RLS Query Performance | Slow | 100x faster |
| Workflow Success Rate | N/A | >99% |

### **Cost**

| Item | Current | With Optimization |
|------|---------|-------------------|
| AI Calls/Month | $300 | $210 (30% cache hit) |
| Worker CPU | $100 | $2 (Workflows) |
| Total | $400 | $212 |
| **Savings** | - | **$188/month** |

### **Reliability**

| Metric | Current | Target |
|--------|---------|--------|
| Apify Downtime Impact | 100% (single point of failure) | 0% (future fallback) |
| Stripe Webhook Loss | Possible | 0% (Queue retries) |
| Analysis Job Failure | Lost | Auto-retry per step |

---

## 🚀 DEPLOYMENT PLAN

### **Week 1: Foundation**
```bash
git checkout -b phase-0-foundation
# Apply RLS fixes, validate bindings
npm run deploy
# Test on staging
```

### **Weeks 2-6: Features (Incremental)**
```bash
git checkout -b feature/credits
# Build credits feature
npm test
npm run deploy
# Validate in production

git checkout -b feature/auth
# Build auth feature
# ... repeat for each feature
```

### **Weeks 7-8: Workflows**
```bash
git checkout -b phase-5-workflows
# Add Workflows + DO + Queues
# Update wrangler.toml
npm run deploy
# Monitor for 1 week
```

### **Weeks 9-10: Observability + Testing**
```bash
git checkout -b phase-6-observability
# Add Analytics Engine + Sentry
npm test  # Full test suite
npm run deploy
```

---

## 📈 POST-IMPLEMENTATION

### **Week 11: Production Monitoring**

**Watch for:**
- Analytics Engine dashboard (track AI costs, latency, errors)
- Sentry alerts (should be rare - only critical)
- Workflow success rate (should be >99%)
- Cache hit rate (should increase over time)

### **Week 12: Performance Tuning**

**Optimize based on real data:**
- Which endpoints are slowest? (Analytics Engine)
- Which AI calls cost most? (AI Gateway logs)
- Which workflows fail most? (Workflow introspection)

### **Week 13+: Feature Development**

**With solid foundation, build:**
- New analysis types
- Enhanced business context
- Advanced filtering
- Bulk operations
- Admin dashboards

---

## 🎯 FINAL RATING

### **Current Worker: 4/10**
- Synchronous blocking
- Security holes
- No observability
- Poor structure

### **Target Worker: 95/100**
- Async orchestration
- Defense-in-depth security
- Full observability
- Clean architecture
- Comprehensive testing
- Production-ready

### **Why not 100?**
Nothing is perfect until proven in production. This is a **95/100 foundation** that will evolve to 100/100 through real-world usage.

---

## ✅ READY TO BUILD

**All decisions finalized. All phases defined. All technologies selected.**

**Start with Phase 0 - Week 1 begins now.**

---

**END OF IMPLEMENTATION PLAN**
