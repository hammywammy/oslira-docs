# Dashboard Leads System - Technical Documentation

## Overview
The dashboard leads system fetches and displays user leads with their analysis runs using a full-stack architecture spanning frontend JavaScript, Cloudflare Workers, and Supabase PostgreSQL.

---

## Architecture Flow

```
Frontend (Browser)
    ↓
ApiClient (public/core/api/ApiClient.js)
    ↓
HttpClient (public/core/infrastructure/HttpClient.js)
    ↓
Cloudflare Worker (cloudflare-workers/src/handlers/leads.ts)
    ↓
Database Service (cloudflare-workers/src/services/database.ts)
    ↓
Supabase View (dashboard_leads_view)
    ↓
PostgreSQL Database (leads + runs tables)
```

---

## Component Breakdown

### 1. **Frontend - LeadsAPI.js**
**Location:** `public/core/api/endpoints/LeadsAPI.js`

**Purpose:** High-level API interface for lead operations

**Method:** `fetchDashboardLeads(businessId, limit)`

**How it works:**
```javascript
1. Validates businessId parameter
2. Calls ApiClient.get() with endpoint `/v1/leads/dashboard`
3. Expects response format: { success: true, data: [...] }
4. Returns array of leads with nested runs
5. Caches response for 2 minutes
```

**Response Structure:**
```javascript
[
  {
    lead_id: "uuid",
    username: "instagram_user",
    display_name: "Full Name",
    follower_count: 50000,
    runs: [
      {
        run_id: "uuid",
        analysis_type: "deep",
        overall_score: 85,
        summary_text: "Analysis summary..."
      }
    ]
  }
]
```

---

### 2. **Frontend - ApiClient.js**
**Location:** `public/core/api/ApiClient.js`

**Purpose:** Centralized HTTP request handler with auth, caching, and deduplication

**How it works:**
```javascript
1. Checks cache for existing response (2-minute TTL)
2. Deduplicates in-flight requests to same endpoint
3. Gets fresh auth token from AuthManager
4. Builds request with Authorization header
5. Calls HttpClient.request()
6. Unwraps HttpClient response wrapper
7. Caches successful GET responses
8. Returns clean API response
```

**Key Feature - Response Unwrapping:**
```javascript
// HttpClient returns: { status: 200, data: { success: true, data: [...] } }
const httpResponse = await requestPromise;
const response = httpResponse.data || httpResponse;  // Unwrap to get actual API response
// Now response = { success: true, data: [...] }
```

---

### 3. **Frontend - HttpClient.js**
**Location:** `public/core/infrastructure/HttpClient.js`

**Purpose:** Low-level HTTP wrapper with retry logic and error handling

**How it works:**
```javascript
1. Receives URL and options from ApiClient
2. Creates AbortController for timeout handling
3. Executes fetch() with configured options
4. Retries failed requests (up to 3 times with exponential backoff)
5. Parses response based on Content-Type
6. Wraps response in standard format:
   {
     status: 200,
     statusText: 'OK',
     headers: Headers,
     data: <parsed response body>
   }
```

---

### 4. **Backend - Cloudflare Worker Endpoint**
**Location:** `cloudflare-workers/src/handlers/leads.ts`

**Function:** `handleGetDashboardLeads()`

**How it works:**
```javascript
1. Generates unique requestId for logging
2. Authenticates request via JWT token
3. Validates business_id query parameter
4. Extracts limit parameter (default 50)
5. Calls getDashboardLeads() from database service
6. Returns standardized response:
   {
     success: true,
     data: [...leads...],
     requestId: "abc123"
   }
7. Handles errors with 500 status and error details
```

**Authentication Flow:**
```javascript
- Extracts Bearer token from Authorization header
- Validates JWT using Supabase client
- Verifies user exists and token is valid
- Returns userId for database queries
```

---

### 5. **Backend - Database Service**
**Location:** `cloudflare-workers/src/services/database.ts`

**Function:** `getDashboardLeads(user_id, business_id, env, limit)`

**How it works:**
```javascript
1. Retrieves Supabase URL and service role key from config
2. Builds REST API query to dashboard_leads_view
3. Filters by user_id and business_id
4. Limits results to specified count
5. Fetches from Supabase REST API
6. Receives leads with pre-joined runs data
7. Sorts runs by created_at descending (client-side)
8. Limits to 5 most recent runs per lead
9. Sorts leads by most recent run date
10. Returns processed array
```

**Query Structure:**
```
GET /rest/v1/dashboard_leads_view?user_id=eq.{id}&business_id=eq.{id}&limit={n}
```

---

### 6. **Database - Supabase View**
**Location:** Supabase SQL Editor

**View:** `dashboard_leads_view`

**How it works:**
```sql
1. Main query selects from leads table
2. LEFT JOIN with CTE that aggregates runs per lead
3. Runs are aggregated into JSON array using json_agg()
4. Each run includes: run_id, analysis_type, scores, summary, timestamps
5. Only most recent 5 runs per lead via ORDER BY + GROUP BY
6. Results ordered by latest run date (NULLS LAST for leads without runs)
7. Indexed on (user_id, business_id, first_discovered_at) for performance
```

**Why a View?**
- Pre-optimized query execution
- Single source of truth for dashboard data
- Better query plan caching in Postgres
- Easier to maintain than complex REST queries
- Consistent join logic across all consumers

---

## Data Models

### Leads Table
```sql
leads (
  lead_id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  business_id UUID NOT NULL,
  username TEXT NOT NULL,
  display_name TEXT,
  profile_picture_url TEXT,
  follower_count INTEGER,
  following_count INTEGER,
  post_count INTEGER,
  is_verified_account BOOLEAN,
  first_discovered_at TIMESTAMP,
  last_updated_at TIMESTAMP,
  UNIQUE(user_id, username, business_id)
)
```

### Runs Table
```sql
runs (
  run_id UUID PRIMARY KEY,
  lead_id UUID REFERENCES leads(lead_id),
  user_id UUID NOT NULL,
  business_id UUID NOT NULL,
  analysis_type TEXT, -- 'light', 'deep', 'xray'
  overall_score INTEGER,
  niche_fit_score INTEGER,
  engagement_score INTEGER,
  summary_text TEXT,
  confidence_level DECIMAL,
  created_at TIMESTAMP,
  FOREIGN KEY (lead_id) REFERENCES leads(lead_id) ON DELETE CASCADE
)
```

---

## Request/Response Lifecycle

### Complete Flow Example:

**1. User loads dashboard**
```javascript
DashboardApp.loadDashboardData()
  → LeadManager.loadDashboardData()
  → LeadsAPI.fetchDashboardLeads(businessId, 50)
```

**2. Frontend API call**
```javascript
ApiClient.get('/v1/leads/dashboard?business_id=xxx&limit=50')
  → Check cache (miss)
  → Get auth token from AuthManager
  → HttpClient.request('https://api.oslira.com/v1/leads/dashboard?...')
```

**3. HTTP request**
```http
GET /v1/leads/dashboard?business_id=717275c8-xxx&limit=50
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**4. Worker processing**
```javascript
handleGetDashboardLeads()
  → Authenticate JWT token
  → Extract user_id from token
  → Validate business_id parameter
  → getDashboardLeads(user_id, business_id, env, 50)
```

**5. Database query**
```javascript
fetch('https://xxx.supabase.co/rest/v1/dashboard_leads_view?...')
  → PostgreSQL executes view
  → Joins leads + runs tables
  → Aggregates runs as JSON
  → Returns filtered results
```

**6. Response unwrapping**
```javascript
HttpClient returns:
{
  status: 200,
  data: { success: true, data: [leads], requestId: "..." }
}

ApiClient unwraps:
response = httpResponse.data  // { success: true, data: [leads] }

LeadsAPI receives:
{ success: true, data: [leads] }

LeadsAPI returns:
[leads array]
```

**7. UI rendering**
```javascript
LeadManager receives leads array
  → Updates StateManager with leads
  → Emits 'leads:loaded' event
  → LeadDisplayUseCase renders table
  → StatsCalculator updates metrics
```

---

## Caching Strategy

### Frontend Cache (ApiClient)
- **Duration:** 2 minutes (120,000ms)
- **Key:** `GET:/v1/leads/dashboard?business_id=xxx&limit=50:`
- **Invalidation:** TTL expiration or manual clearCachePattern()
- **Storage:** In-memory Map

### Why 2 Minutes?
- Balances freshness with performance
- Prevents duplicate requests during page interactions
- Allows real-time updates via EventBus without staleness

---

## Error Handling

### Frontend Errors
```javascript
try {
  const leads = await leadsAPI.fetchDashboardLeads(businessId);
} catch (error) {
  // Error types:
  // - 'Authentication token not available'
  // - 'Failed to fetch dashboard leads'
  // - 'Unexpected response format from API'
  // - Network errors from HttpClient
}
```

### Backend Errors
```javascript
// Worker returns:
{
  success: false,
  error: "Dashboard query failed: 400",
  requestId: "abc123"
}

// With status codes:
401 - Unauthorized (invalid/missing JWT)
400 - Bad Request (missing business_id)
500 - Internal Server Error (database/query failure)
```

---

## Performance Optimizations

### Database Level
1. **View pre-aggregation** - Complex joins computed once
2. **Composite indexes** - Fast filtering on (user_id, business_id)
3. **Limited runs per lead** - Only 5 most recent to reduce payload
4. **JSONB aggregation** - Efficient nested data structure

### Worker Level
1. **Connection pooling** - Reuses Supabase connections
2. **Config caching** - Stores env vars in memory
3. **Minimal transformations** - Returns view data directly
4. **Structured logging** - Fast CloudWatch insights

### Frontend Level
1. **Request deduplication** - Prevents duplicate in-flight calls
2. **Response caching** - Avoids redundant API calls
3. **Virtual scrolling** - Renders only visible table rows
4. **Optimistic updates** - UI responds before API confirmation

---

## Monitoring & Debugging

### Logging Points

**Frontend:**
```javascript
[LeadsAPI] Fetching dashboard leads for business: xxx
[ApiClient] Request successful (200ms)
[LeadManager] Loaded 15 leads
```

**Backend:**
```json
{
  "level": "info",
  "message": "Fetching dashboard leads",
  "userId": "3281cd45-xxx",
  "businessId": "717275c8-xxx",
  "limit": 50
}
```

### Key Metrics
- **Request duration:** Target <200ms end-to-end
- **Cache hit rate:** Should be >60% for dashboard loads
- **Error rate:** Target <1% of requests
- **Database query time:** <50ms for view query

---

## Security

### Authentication
- JWT token required for all requests
- Token validated against Supabase auth
- User ID extracted from verified token
- Business ID ownership verified in query filter

### Authorization
- Row-level security via user_id filter
- Users can only see their own leads
- Business_id scoped to user's profiles
- No cross-tenant data leakage

### Data Protection
- Service role key stored in AWS Secrets Manager
- Frontend uses anon key (limited permissions)
- CORS configured for app.oslira.com only
- All connections over HTTPS/TLS

---

## Future Enhancements

### Potential Improvements
1. **Redis caching layer** - Shared cache across workers
2. **GraphQL endpoint** - Client-specified fields
3. **Pagination** - Cursor-based for large datasets
4. **Real-time subscriptions** - WebSocket updates for new leads
5. **Materialized view** - Pre-computed for faster queries
6. **CDN caching** - Edge-cached responses for common queries
