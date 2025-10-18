# 🚀 Oslira API Documentation

**Version:** v4.0.0-enterprise  
**Base URL:** `https://api.oslira.com`  
**Architecture:** Resource-based REST API  
**Authentication:** JWT Bearer tokens (Supabase Auth)

---

## 📋 Quick Reference - All Endpoints

### **User Resources**
```
GET    /user/profile              → Get current user info
PATCH  /user/profile              → Update user profile
GET    /user/subscription         → Get subscription & credits
GET    /user/usage                → Get current month usage
GET    /user/usage/history        → Get historical usage
```

### **Credits Resources**
```
GET    /credits/balance           → Get credit balance (lightweight)
GET    /credits/transactions      → Get transaction history (paginated)
```

### **Business Resources**
```
GET    /business-profiles         → List user's businesses
POST   /business-profiles         → Create new business
GET    /business-profiles/:id     → Get single business
PATCH  /business-profiles/:id     → Update business
DELETE /business-profiles/:id     → Delete business
```

### **Leads Resources**
```
GET    /leads                     → List leads (flexible filtering)
GET    /leads/:lead_id            → Get lead details
GET    /leads/:lead_id/runs       → Get lead's analysis history
GET    /v1/leads/dashboard        → Dashboard view (legacy)
```

### **Analysis (Runs) Resources**
```
GET    /runs/:run_id              → Get analysis overview
GET    /runs/:run_id/payload      → Get full AI payload (lazy-load)
POST   /v1/analyze                → Run single analysis
POST   /v1/bulk-analyze           → Run batch analysis
```

---

## 🔐 Authentication

All endpoints (except `/config`) require a JWT token from Supabase Auth.

### **How to Authenticate**

```javascript
// Get token from Supabase
const { data: { session } } = await supabase.auth.getSession();
const token = session.access_token;

// Use in API calls
const response = await fetch('https://api.oslira.com/user/profile', {
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  }
});
```

---

## 📖 Endpoint Details

---

## **1. User Endpoints**

### **GET /user/profile**
Get current user's profile information.

**Request:**
```bash
curl -X GET https://api.oslira.com/user/profile \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "email": "user@example.com",
    "full_name": "John Doe",
    "signature_name": "John D.",
    "created_at": "2025-01-10T12:00:00Z",
    "onboarding_completed": true
  },
  "request_id": "req_abc123"
}
```

---

### **PATCH /user/profile**
Update current user's profile.

**Request:**
```bash
curl -X PATCH https://api.oslira.com/user/profile \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "full_name": "John Doe Updated",
    "signature_name": "JD"
  }'
```

**Response:**
```json
{
  "success": true,
  "data": {
    "updated": true
  },
  "message": "Profile updated successfully",
  "request_id": "req_xyz789"
}
```

---

### **GET /user/subscription**
Get subscription details and credit balance.

**Request:**
```bash
curl -X GET https://api.oslira.com/user/subscription \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "plan_type": "pro",
    "credits_remaining": 450,
    "status": "active",
    "current_period_start": "2025-01-01T00:00:00Z",
    "current_period_end": "2025-02-01T00:00:00Z",
    "stripe_customer_id": "cus_xxx",
    "stripe_subscription_id": "sub_xxx"
  },
  "request_id": "req_def456"
}
```

---

### **GET /user/usage**
Get current month's usage statistics.

**Request:**
```bash
curl -X GET https://api.oslira.com/user/usage \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "month": "2025-01-01",
    "leads_researched": 12,
    "credits_used": 18,
    "analyses": {
      "light": 5,
      "deep": 7,
      "xray": 0,
      "single": 10,
      "bulk": 2
    },
    "avg_lead_score": 78,
    "premium_leads_found": 3
  },
  "request_id": "req_ghi789"
}
```

---

### **GET /user/usage/history**
Get historical usage statistics (paginated).

**Request:**
```bash
curl -X GET "https://api.oslira.com/user/usage/history?limit=12&offset=0" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "history": [
      {
        "month": "2025-01-01",
        "leads_researched": 12,
        "credits_used": 18,
        "analyses": { "light": 5, "deep": 7, "xray": 0 },
        "avg_lead_score": 78,
        "premium_leads_found": 3
      }
    ],
    "pagination": {
      "limit": 12,
      "offset": 0,
      "total": 6,
      "hasMore": false
    }
  },
  "request_id": "req_jkl012"
}
```

---

## **2. Credits Endpoints**

### **GET /credits/balance**
Get current credit balance (lightweight, for real-time updates).

**Request:**
```bash
curl -X GET https://api.oslira.com/credits/balance \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "credits_remaining": 450,
    "plan_type": "pro"
  },
  "request_id": "req_mno345"
}
```

**Use Case:** Call this after each analysis to update header display.

---

### **GET /credits/transactions**
Get paginated credit transaction history.

**Request:**
```bash
curl -X GET "https://api.oslira.com/credits/transactions?limit=50&offset=0&type=use" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Query Parameters:**
- `limit` (optional): Number of records (default: 50)
- `offset` (optional): Pagination offset (default: 0)
- `type` (optional): Filter by type: `use`, `purchase`, `refund`

**Response:**
```json
{
  "success": true,
  "data": {
    "transactions": [
      {
        "id": "uuid",
        "amount": -2,
        "type": "use",
        "description": "deep analysis",
        "created_at": "2025-01-15T10:30:00Z",
        "run_id": "uuid",
        "details": {
          "actual_cost": 0.0045,
          "tokens_in": 1200,
          "tokens_out": 800,
          "model_used": "gpt-4o"
        }
      }
    ],
    "summary": {
      "total_used": 18,
      "total_purchased": 500
    },
    "pagination": {
      "limit": 50,
      "offset": 0,
      "total": 18,
      "hasMore": false
    }
  },
  "request_id": "req_pqr678"
}
```

---

## **3. Business Endpoints**

### **GET /business-profiles**
List all business profiles for current user.

**Request:**
```bash
curl -X GET https://api.oslira.com/business-profiles \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "business_name": "My Agency",
      "business_niche": "Email Marketing",
      "target_audience": "B2B SaaS Companies",
      "business_one_liner": "We help SaaS scale email revenue",
      "monthly_lead_goal": 50,
      "created_at": "2025-01-05T08:00:00Z"
    }
  ],
  "request_id": "req_stu901"
}
```

---

### **POST /business-profiles**
Create a new business profile.

**Request:**
```bash
curl -X POST https://api.oslira.com/business-profiles \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "business_name": "My Agency",
    "business_niche": "Email Marketing",
    "target_audience": "B2B SaaS Companies",
    "value_proposition": "Increase email revenue by 3x",
    "monthly_lead_goal": 50
  }'
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "business_name": "My Agency",
    "created_at": "2025-01-15T12:00:00Z"
  },
  "request_id": "req_vwx234"
}
```

---

### **GET /business-profiles/:id**
Get single business profile details.

**Request:**
```bash
curl -X GET https://api.oslira.com/business-profiles/{business_id} \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "business_name": "My Agency",
    "business_niche": "Email Marketing",
    "target_audience": "B2B SaaS Companies",
    "business_one_liner": "We help SaaS scale email revenue",
    "business_context_pack": { /* AI-generated context */ },
    "monthly_lead_goal": 50,
    "created_at": "2025-01-05T08:00:00Z"
  },
  "request_id": "req_yza567"
}
```

---

### **PATCH /business-profiles/:id**
Update business profile.

**Request:**
```bash
curl -X PATCH https://api.oslira.com/business-profiles/{business_id} \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "business_name": "My Agency Updated",
    "monthly_lead_goal": 100
  }'
```

**Response:**
```json
{
  "success": true,
  "data": {
    "updated": true
  },
  "message": "Profile updated successfully",
  "request_id": "req_bcd890"
}
```

---

### **DELETE /business-profiles/:id**
Soft delete business profile.

**Request:**
```bash
curl -X DELETE https://api.oslira.com/business-profiles/{business_id} \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "deleted": true
  },
  "message": "Profile deleted successfully",
  "request_id": "req_efg123"
}
```

---

## **4. Leads Endpoints**

### **GET /leads**
List leads with flexible filtering and sorting.

**Request:**
```bash
curl -X GET "https://api.oslira.com/leads?business_id={business_id}&limit=50&offset=0&sort_by=follower_count&sort_order=desc&min_score=75" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Query Parameters:**
- `business_id` (required): Business profile ID
- `limit` (optional): Number of records (default: 50)
- `offset` (optional): Pagination offset (default: 0)
- `sort_by` (optional): `created_at`, `follower_count`, `username`
- `sort_order` (optional): `asc`, `desc` (default: desc)
- `min_score` (optional): Filter by minimum analysis score

**Response:**
```json
{
  "success": true,
  "data": {
    "leads": [
      {
        "lead_id": "uuid",
        "username": "johndoe",
        "display_name": "John Doe",
        "profile_picture_url": "https://...",
        "bio_text": "Entrepreneur | Marketing Expert",
        "follower_count": 50000,
        "is_verified": true,
        "first_discovered_at": "2025-01-10T10:00:00Z",
        "latest_analysis": {
          "run_id": "uuid",
          "analysis_type": "deep",
          "overall_score": 87,
          "niche_fit_score": 85,
          "engagement_score": 90,
          "summary": "High engagement, excellent fit",
          "analyzed_at": "2025-01-15T12:00:00Z"
        }
      }
    ],
    "pagination": {
      "limit": 50,
      "offset": 0,
      "total": 120,
      "hasMore": true
    }
  },
  "request_id": "req_hij456"
}
```

---

### **GET /leads/:lead_id**
Get detailed information about a single lead.

**Request:**
```bash
curl -X GET https://api.oslira.com/leads/{lead_id} \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "lead_id": "uuid",
    "username": "johndoe",
    "display_name": "John Doe",
    "profile_picture_url": "https://...",
    "bio_text": "Entrepreneur | Marketing Expert",
    "external_website_url": "https://johndoe.com",
    "profile_url": "https://instagram.com/johndoe",
    "follower_count": 50000,
    "following_count": 1200,
    "post_count": 450,
    "is_verified_account": true,
    "is_private_account": false,
    "is_business_account": true,
    "first_discovered_at": "2025-01-10T10:00:00Z",
    "last_updated_at": "2025-01-15T12:00:00Z",
    "latest_analysis": {
      "run_id": "uuid",
      "analysis_type": "deep",
      "overall_score": 87,
      "niche_fit_score": 85,
      "engagement_score": 90,
      "summary": "High engagement, excellent fit",
      "confidence_level": 0.92,
      "analyzed_at": "2025-01-15T12:00:00Z"
    },
    "analysis_stats": {
      "total_analyses": 3,
      "by_type": {
        "light": 1,
        "deep": 2,
        "xray": 0
      }
    }
  },
  "request_id": "req_klm789"
}
```

---

### **GET /leads/:lead_id/runs**
Get all analysis runs for a specific lead (analysis history).

**Request:**
```bash
curl -X GET "https://api.oslira.com/leads/{lead_id}/runs?limit=20&offset=0" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "runs": [
      {
        "run_id": "uuid",
        "analysis_type": "deep",
        "overall_score": 87,
        "niche_fit_score": 85,
        "engagement_score": 90,
        "summary": "High engagement, excellent fit",
        "confidence_level": 0.92,
        "status": "completed",
        "ai_model_used": "gpt-4o",
        "created_at": "2025-01-15T12:00:00Z",
        "completed_at": "2025-01-15T12:02:30Z",
        "processing_time_ms": 150000
      }
    ],
    "stats": {
      "total_runs": 3,
      "completed_runs": 3,
      "avg_score": 84,
      "score_history": [
        { "date": "2025-01-15T12:00:00Z", "score": 87, "type": "deep" },
        { "date": "2025-01-12T10:00:00Z", "score": 82, "type": "light" }
      ]
    },
    "pagination": {
      "limit": 20,
      "offset": 0,
      "total": 3,
      "hasMore": false
    }
  },
  "request_id": "req_nop012"
}
```

---

## **5. Runs (Analysis) Endpoints**

### **GET /runs/:run_id**
Get analysis overview (without full payload).

**Request:**
```bash
curl -X GET https://api.oslira.com/runs/{run_id} \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "run_id": "uuid",
    "analysis_type": "deep",
    "analysis_version": "v4.0",
    "status": "completed",
    "overall_score": 87,
    "niche_fit_score": 85,
    "engagement_score": 90,
    "confidence_level": 0.92,
    "summary": "High engagement, excellent fit for your niche",
    "ai_model_used": "gpt-4o",
    "created_at": "2025-01-15T12:00:00Z",
    "started_at": "2025-01-15T12:00:05Z",
    "completed_at": "2025-01-15T12:02:30Z",
    "processing_time_ms": 145000,
    "lead": {
      "lead_id": "uuid",
      "username": "johndoe",
      "display_name": "John Doe",
      "profile_picture_url": "https://...",
      "follower_count": 50000,
      "is_verified": true
    },
    "has_full_payload": true
  },
  "request_id": "req_qrs345"
}
```

---

### **GET /runs/:run_id/payload**
Get full AI-generated analysis payload (large data, lazy-loaded).

**⚠️ Use Case:** Only call this when user explicitly clicks "View Full Analysis" to avoid unnecessary data transfer.

**Request:**
```bash
curl -X GET https://api.oslira.com/runs/{run_id}/payload \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "run_id": "uuid",
    "analysis_type": "deep",
    "data_size_bytes": 52000,
    "created_at": "2025-01-15T12:02:30Z",
    "payload": {
      "deep_summary": "Detailed analysis...",
      "selling_points": ["Point 1", "Point 2"],
      "outreach_message": "Hi John, I noticed...",
      "engagement_breakdown": {
        "avg_likes": 2500,
        "avg_comments": 150,
        "engagement_rate": 5.2
      },
      "latest_posts": [ /* array of posts */ ],
      "audience_insights": { /* detailed insights */ },
      "personality_profile": { /* AI-generated profile */ }
    }
  },
  "request_id": "req_tuv678"
}
```

---

## 🏗️ Architecture Overview

### **Design Principles**
1. **Resource-Oriented REST** - Each resource has its own endpoint family
2. **Granular Caching** - Different TTLs per resource type
3. **Lazy Loading** - Heavy data (payloads) loaded on-demand
4. **Stateless** - Cloudflare Workers, no server-side sessions
5. **JWT Auth** - Supabase handles authentication

### **File Structure**
```
src/api/
├── user/               → User profile, subscription, usage
├── credits/            → Balance and transactions
├── business/           → Business profiles CRUD
├── leads/              → Lead management and querying
├── runs/               → Analysis results and payloads
├── analysis/           → Analysis execution (POST actions)
├── billing/            → Stripe webhooks
├── admin/              → Admin panel operations
└── config/             → Public configuration
```

### **Database Tables**
- `users` - User accounts
- `subscriptions` - Plans and credits
- `business_profiles` - User's business contexts
- `leads` - Instagram profiles
- `runs` - Analysis executions
- `payloads` - Full AI-generated data
- `credit_transactions` - Credit usage history
- `usage_tracking` - Monthly usage aggregates

---

## 🔄 Common Workflows

### **Dashboard Page Load**
```javascript
// 1. Load user data (parallel)
const [profile, subscription, usage] = await Promise.all([
  fetch('/user/profile'),
  fetch('/user/subscription'),
  fetch('/user/usage')
]);

// 2. Load business list
const businesses = await fetch('/business-profiles');

// 3. Load leads for selected business
const leads = await fetch(`/leads?business_id=${selectedBusinessId}`);
```

### **After Analysis Completes**
```javascript
// 1. Refresh credit balance only (lightweight)
const { credits_remaining } = await fetch('/credits/balance');

// 2. Refresh leads list for that business
const leads = await fetch(`/leads?business_id=${businessId}`);
```

### **View Lead Details**
```javascript
// 1. Load lead overview
const lead = await fetch(`/leads/${leadId}`);

// 2. Load analysis history
const { runs } = await fetch(`/leads/${leadId}/runs`);

// 3. When user clicks "View Full Analysis"
const { payload } = await fetch(`/runs/${runId}/payload`);
```

---

## 📊 Response Format

All endpoints follow this standard format:

**Success Response:**
```json
{
  "success": true,
  "data": { /* resource data */ },
  "message": "Optional success message",
  "request_id": "req_xxx"
}
```

**Error Response:**
```json
{
  "success": false,
  "data": null,
  "error": "Error message",
  "request_id": "req_xxx"
}
```

---

## 🚨 Error Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad Request (invalid parameters) |
| 401 | Unauthorized (missing/invalid token) |
| 404 | Not Found (resource doesn't exist) |
| 500 | Internal Server Error |

---

## 🎯 Next Steps

1. **Frontend Integration** - Use these endpoints in your React app
2. **Caching Strategy** - Implement React Query with appropriate stale times
3. **Error Handling** - Handle 401s with token refresh
4. **Optimistic Updates** - Update UI before API confirms

---

**Last Updated:** 2025-01-18  
**Maintained By:** Oslira Engineering Team
