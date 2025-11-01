# **OSLIRA CREDIT, BILLING, USAGE & FEATURES SYSTEM V2**
## **Final Production Implementation Plan**

---

## **EXECUTIVE SUMMARY**

This document defines the complete credit-based billing system for Oslira with refined pricing that prevents exploitation while maximizing market capture. The system targets 95%+ gross margins with pricing from $0-$999/month, designed for immediate implementation without enterprise features (those come later when earned).

---

## **1. FINALIZED PRICING STRUCTURE**

### **1.1 Tier Breakdown**

| Tier | Monthly Price | Credits | Light Analyses | Target Customer | Priority |
|------|--------------|---------|----------------|-----------------|----------|
| **Free** | $0 | 10 | 10 | Trial/Testing | 4 (Lowest) |
| **Growth** | $29 | 250 | 250 | Solopreneurs | 3 |
| **Pro** | $99 | 1,500 | 1,500 | Growing Businesses | 2 |
| **Agency** | $299 | 5,000 | 5,000 | Agencies/Teams | 2 |
| **Enterprise** | $999 | 20,000 | 20,000 | Scale Operations | 1 (Highest) |

### **1.2 Anti-Gaming Mechanics**

**Free Tier (10+10)**:
- Creating 3 accounts for 30 credits = 3 separate onboardings
- Not worth the effort for $29 value
- Just enough to validate quality

**Value Progression**:
```
Growth: $0.116 per credit
Pro: $0.066 per credit (43% better)
Agency: $0.060 per credit (9% better)
Enterprise: $0.050 per credit (17% better)
```

**Gaming Prevention**: Each tier provides better value per credit, eliminating incentive for multiple accounts

---

## **2. CREDIT SYSTEM MECHANICS**

### **2.1 Credit Definition**

**Credits** = Currency for Deep and X-Ray analyses
- **Allocation**: Monthly grant based on subscription tier
- **Reset**: Aligned with Stripe billing period (NOT calendar month)
- **Rollover**: None - use or lose each period
- **Expiration**: End of billing period

**Light Analyses** = Separate quota system
- **Cost**: Free within monthly quota
- **Overflow**: Cannot perform after quota exhausted (no credit option)
- **Reset**: Same as credits

### **2.2 Analysis Costs**

| Analysis Type | Credit Cost | Actual Cost (Estimated) | Margin |
|--------------|-------------|------------------------|---------|
| **Light** | 0 | $0.001 | N/A (free) |
| **Deep** | 1 | $0.01 | 99% |
| **X-Ray** | 2 | $0.01 | 99.5% |

**Note**: Must validate actual costs with 100 test runs, then update `analysis_costs` table

---

## **3. BILLING IMPLEMENTATION**

### **3.1 Payment Processing Flow**

```javascript
// Stripe Webhook Events

"invoice.payment_succeeded" → {
  if (billing_reason === 'subscription_cycle') {
    resetCredits(accountId, tier);
    createNewBillingPeriod();
  }
}

"invoice.payment_failed" → {
  immediateDowngradeToFree(accountId);
  sendNotification('payment_failed');
}

"customer.subscription.updated" → {
  if (upgrade) {
    grantDifferentialCredits();
    chargeProrated();
  }
  if (downgrade) {
    scheduleForNextPeriod();
  }
}
```

### **3.2 Upgrade Credit Calculation**

**Method: Differential Credits** (Prevents Gaming)

```javascript
function calculateUpgradeCredits(oldTier, newTier, accountId) {
  const oldCredits = TIERS[oldTier].credits;
  const newCredits = TIERS[newTier].credits;
  
  // Only grant the DIFFERENCE
  const creditsToGrant = newCredits - oldCredits;
  
  // Log for abuse monitoring
  await logCreditGrant({
    accountId,
    amount: creditsToGrant,
    type: 'upgrade',
    oldTier,
    newTier
  });
  
  return creditsToGrant;
}
```

---

## **4. QUEUE & CONCURRENCY MANAGEMENT**

### **4.1 Priority Queue System**

```javascript
const QUEUE_PRIORITY = {
  enterprise: 1,  // First
  agency: 2,      // Second
  pro: 2,         // Second (same as Agency)
  growth: 3,      // Third
  free: 4         // Last
};

const CONCURRENT_LIMITS = {
  free: 1,
  growth: 2,
  pro: 5,
  agency: 10,
  enterprise: 20  // Limited by Apify's 10 anyway
};
```

### **4.2 Rate Limiting**

**Hourly Limits for Light Analyses**:
```javascript
const LIGHT_RATE_LIMITS = {
  free: 5,        // Half their daily quota
  growth: 25,     // 10% of monthly
  pro: 150,       // 10% of monthly
  agency: 500,    // 10% of monthly
  enterprise: 2000 // 10% of monthly
};
```

**Bulk Upload Limits**:
```javascript
const BULK_LIMITS = {
  free: 5,
  growth: 25,
  pro: 100,
  agency: 250,
  enterprise: 500
};
```

---

## **5. DATABASE SCHEMA**

### **5.1 Core Tables**

```sql
-- Dynamic pricing configuration
CREATE TABLE pricing_tiers (
  tier_name VARCHAR(20) PRIMARY KEY,
  price DECIMAL(10,2),
  credits INTEGER,
  light_quota INTEGER,
  light_rate_limit INTEGER,
  concurrent_slots INTEGER,
  queue_priority INTEGER,
  bulk_max INTEGER,
  features JSONB DEFAULT '{}',
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Initial data
INSERT INTO pricing_tiers VALUES
  ('free', 0, 10, 10, 5, 1, 4, 5, '{"support":["faq"]}', true),
  ('growth', 29, 250, 250, 25, 2, 3, 25, '{"support":["faq","email"]}', true),
  ('pro', 99, 1500, 1500, 150, 5, 2, 100, '{"support":["email","priority"]}', true),
  ('agency', 299, 5000, 5000, 500, 10, 2, 250, '{"support":["priority"]}', true),
  ('enterprise', 999, 20000, 20000, 2000, 20, 1, 500, '{"support":["dedicated"]}', true);

-- Subscription tracking
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID REFERENCES accounts(id),
  stripe_subscription_id VARCHAR(255) UNIQUE,
  stripe_customer_id VARCHAR(255),
  tier VARCHAR(20) REFERENCES pricing_tiers(tier_name),
  status VARCHAR(50),
  current_period_start TIMESTAMP,
  current_period_end TIMESTAMP,
  credits_remaining INTEGER DEFAULT 0,
  light_remaining INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Usage tracking per billing period
CREATE TABLE usage_tracking (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID REFERENCES accounts(id),
  billing_period_start TIMESTAMP,
  billing_period_end TIMESTAMP,
  light_used INTEGER DEFAULT 0,
  light_quota INTEGER,
  credits_used DECIMAL(10,1) DEFAULT 0,
  credits_allocated INTEGER,
  deep_count INTEGER DEFAULT 0,
  xray_count INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(account_id, billing_period_start)
);

-- Credit grant audit trail
CREATE TABLE credit_grants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID REFERENCES accounts(id),
  amount INTEGER,
  grant_type VARCHAR(50), -- 'monthly_reset', 'upgrade', 'downgrade'
  old_tier VARCHAR(20),
  new_tier VARCHAR(20),
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Analysis costs (dynamic)
CREATE TABLE analysis_costs (
  analysis_type VARCHAR(20) PRIMARY KEY,
  credits_required DECIMAL(3,1),
  actual_cost_cents DECIMAL(10,4),
  last_validated TIMESTAMP,
  notes TEXT
);

INSERT INTO analysis_costs VALUES
  ('light', 0, 0.1, NULL, 'Free within quota'),
  ('deep', 1.0, 1.0, NULL, 'Update after testing'),
  ('xray', 2.0, 1.0, NULL, 'Update after testing');
```

---

## **6. CONFIGURATION SYSTEM**

### **6.1 Centralized Configuration**

```typescript
// src/config/pricing.config.ts
export const PRICING_CONFIG = {
  tiers: {
    free: {
      price: 0,
      credits: 10,
      light_quota: 10,
      light_rate_limit: 5,
      concurrent_slots: 1,
      queue_priority: 4,
      bulk_max: 5,
      features: {
        support: ['faq'],
        data_export: ['csv'],
        analysis_history_days: 30
      }
    },
    growth: {
      price: 29,
      credits: 250,
      light_quota: 250,
      light_rate_limit: 25,
      concurrent_slots: 2,
      queue_priority: 3,
      bulk_max: 25,
      features: {
        support: ['faq', 'email'],
        data_export: ['csv'],
        analysis_history_days: 60
      }
    },
    pro: {
      price: 99,
      credits: 1500,
      light_quota: 1500,
      light_rate_limit: 150,
      concurrent_slots: 5,
      queue_priority: 2,
      bulk_max: 100,
      features: {
        support: ['email', 'priority'],
        data_export: ['csv', 'json'],
        analysis_history_days: 180
      }
    },
    agency: {
      price: 299,
      credits: 5000,
      light_quota: 5000,
      light_rate_limit: 500,
      concurrent_slots: 10,
      queue_priority: 2,
      bulk_max: 250,
      features: {
        support: ['priority'],
        data_export: ['csv', 'json'],
        analysis_history_days: 365
      }
    },
    enterprise: {
      price: 999,
      credits: 20000,
      light_quota: 20000,
      light_rate_limit: 2000,
      concurrent_slots: 20,
      queue_priority: 1,
      bulk_max: 500,
      features: {
        support: ['dedicated', 'slack'],
        data_export: ['csv', 'json', 'api'],
        analysis_history_days: -1 // unlimited
      }
    }
  },
  
  credit_costs: {
    light: 0,
    deep: 1,
    xray: 2
  },
  
  system: {
    apify_total_slots: 10,
    max_queue_depth: 1000,
    alert_wait_time_ms: 180000,
    rate_limit_window_ms: 3600000
  }
};
```

---

## **7. LIGHT ANALYSIS ADD-ONS**

### **7.1 Credit Pack Options** (Future Implementation)

```javascript
const LIGHT_PACKS = {
  boost: {
    price: 4.99,
    analyses: 100,
    description: "Quick boost for campaigns"
  },
  power: {
    price: 19.99,
    analyses: 500,
    description: "Major campaign support"
  },
  bulk: {
    price: 79.99,
    analyses: 2500,
    description: "Enterprise projects"
  }
};

// Implementation note: Credits expire with billing period
// No rollover complexity needed
```

---

## **8. MONITORING REQUIREMENTS**

### **8.1 Critical Metrics**

```javascript
const MONITORING_CONFIG = {
  // Alert thresholds
  alerts: {
    queue_depth: 100,        // Alert if exceeded
    avg_wait_time: 180000,   // Alert if > 3 min
    apify_slots_used: 8,     // Alert if consistently at limit
    payment_failure_rate: 0.05, // Alert if > 5%
    cost_per_deep: 0.02,     // Alert if exceeds estimate
  },
  
  // Track hourly
  hourly_metrics: [
    'queue_depth',
    'active_apify_slots',
    'avg_wait_time_ms',
    'analyses_per_hour',
    'error_rate'
  ],
  
  // Track daily
  daily_metrics: [
    'actual_cost_per_type',
    'credits_consumed',
    'light_quota_usage',
    'upgrade_count',
    'downgrade_count',
    'mrr_by_tier'
  ],
  
  // Business KPIs (weekly)
  weekly_kpis: [
    'free_to_paid_conversion',  // Target: >5%
    'growth_to_pro_conversion',  // Target: >30%
    'credit_utilization',        // Target: 60-80%
    'gross_margin',              // Target: >95%
    'churn_by_tier'
  ]
};
```

---

## **9. IMPLEMENTATION TIMELINE**

### **Week 1: Foundation**

**Day 1-2: Database & Configuration**
- [ ] Create all tables from Section 5
- [ ] Implement pricing.config.ts
- [ ] Migrate existing subscription data
- [ ] Set up test environment

**Day 3-4: Stripe Integration**
```javascript
// Key webhooks to implement
app.post('/webhooks/stripe', async (req, res) => {
  const event = req.body;
  
  switch(event.type) {
    case 'invoice.payment_succeeded':
      await handleRenewal(event);
      break;
    case 'invoice.payment_failed':
      await handleDowngrade(event);
      break;
    case 'customer.subscription.updated':
      await handleUpgrade(event);
      break;
  }
});
```

**Day 5: Credit System Core**
- [ ] Credit allocation logic
- [ ] Differential grant calculation
- [ ] Usage tracking implementation

### **Week 2: Usage Management**

**Day 1-2: Queue System**
- [ ] Priority queue implementation
- [ ] Concurrent limits per tier
- [ ] Queue monitoring

**Day 3-4: Rate Limiting**
```javascript
async function enforceRateLimit(accountId, tier, type) {
  const key = `${type}_rate:${accountId}:${getCurrentHour()}`;
  const limit = PRICING_CONFIG.tiers[tier][`${type}_rate_limit`];
  
  const current = await redis.incr(key);
  await redis.expire(key, 3600);
  
  if (current > limit) {
    throw new RateLimitError(`Limit: ${limit}/hour`);
  }
}
```

**Day 5: Dashboard Updates**
- [ ] Credit balance display
- [ ] Light quota display
- [ ] Billing period countdown
- [ ] Usage history

### **Week 3: Analysis Integration**

**Day 1-2: Cost Validation**
```javascript
// Run this immediately
async function validateCosts() {
  const tests = {
    light: [],
    deep: [],
    xray: []
  };
  
  for (let i = 0; i < 100; i++) {
    for (const type of ['light', 'deep', 'xray']) {
      const cost = await runAnalysisAndMeasureCost(type);
      tests[type].push(cost);
    }
  }
  
  // Update database with real costs
  for (const [type, costs] of Object.entries(tests)) {
    const avgCost = costs.reduce((a,b) => a+b) / costs.length;
    await updateAnalysisCost(type, avgCost);
  }
}
```

**Day 3-4: Queue Integration**
- [ ] Connect analysis system to queue
- [ ] Implement priority processing
- [ ] Add monitoring hooks

**Day 5: Testing**
- [ ] Load test with 100 concurrent users
- [ ] Test upgrade/downgrade flows
- [ ] Verify rate limits work

### **Week 4: Production Launch**

**Day 1-2: Migration**
- [ ] Migrate existing users to new tiers
- [ ] Grandfather current pricing for 30 days
- [ ] Send notification emails

**Day 3-4: Monitoring**
```javascript
// Set up dashboards
const DASHBOARD_PANELS = {
  operations: [
    'Queue Depth',
    'Average Wait Time',
    'Apify Utilization'
  ],
  financial: [
    'MRR by Tier',
    'Credit Utilization',
    'Actual Costs vs Estimates'
  ],
  customer: [
    'Conversions by Tier',
    'Usage Patterns',
    'Churn Analysis'
  ]
};
```

**Day 5: Launch**
- [ ] Deploy to production
- [ ] Monitor for 24 hours
- [ ] Adjust based on data

---

## **10. API ENDPOINTS**

### **10.1 Required Endpoints**

```typescript
// Billing
GET  /api/billing/subscription      // Current subscription
POST /api/billing/upgrade          // Upgrade tier
POST /api/billing/downgrade        // Schedule downgrade
GET  /api/billing/usage           // Current period usage

// Credits
GET  /api/credits/balance         // Current balances
GET  /api/credits/history         // Transaction history

// Analysis
POST /api/analyze                 // Submit analysis
GET  /api/queue/position          // Check queue position
GET  /api/queue/stats            // Queue statistics
```

---

## **11. DECISION LOG**

### **11.1 Final Decisions Made**

1. **No $9 tier** - Prevents gaming, maintains quality
2. **Differential credits on upgrade** - Prevents exploitation
3. **No credit rollover** - Industry standard, simpler
4. **Immediate downgrade on payment failure** - No grace period
5. **10/10 free tier** - Just enough to test
6. **No Light overflow to credits** - Keep systems separate

### **11.2 Future Decisions**

1. **Enterprise features** - Add when >50 customers
2. **Annual plans** - Consider at $10K MRR
3. **API access** - Build when requested
4. **Team seats** - Implement with first agency request

---

## **12. SUCCESS METRICS**

### **Launch Week Goals**
- Zero billing errors
- <3 minute average wait time
- <10 support tickets about pricing
- 95%+ uptime

### **Month 1 Goals**
- 20+ paying customers
- 5% free-to-paid conversion
- 70% credit utilization average
- 95%+ gross margins confirmed

### **Month 3 Goals**
- $5K MRR
- 30% Growth→Pro conversion
- <5% monthly churn
- Apify upgrade needed (good problem)

---

## **13. RISK MITIGATION**

| Risk | Detection | Response |
|------|-----------|----------|
| Gaming via multiple accounts | Same IP, similar emails | Account consolidation |
| Queue overflow | Wait time >5 min | Upgrade Apify |
| Cost overrun | Margin <90% | Adjust credit costs |
| Mass downgrades | >10% downgrade rate | Survey for issues |
| Payment failures | >5% failure rate | Review payment flow |

---

## **APPROVAL CHECKLIST**

- [ ] Approve 5-tier structure ($0/$29/$99/$299/$999)
- [ ] Approve 10/10 free tier
- [ ] Approve differential credit grants
- [ ] Approve no rollover policy
- [ ] Approve immediate downgrade on payment failure
- [ ] Approve database schema
- [ ] Approve 4-week implementation timeline

---

**This plan is production-ready for immediate implementation. All components are designed to work together as a cohesive system that maximizes revenue while preventing exploitation.**
