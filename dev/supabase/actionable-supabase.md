# SUPABASE SCHEMA - ONBOARDING INTEGRATION

## DATABASE STRUCTURE

### 1. users
```sql
id: uuid (PK)
email: text (UNIQUE)
full_name: text
avatar_url: text | null
onboarding_completed: boolean (default: false)
onboarding_completed_at: timestamptz | null
is_admin: boolean (default: false)
last_seen_at: timestamptz
created_at: timestamptz
updated_at: timestamptz
```

**Notes:**
- `signature_name` REMOVED (moved to business_profiles)
- `onboarding_completed_at` tracks completion timestamp

---

### 2. accounts
```sql
id: uuid (PK)
owner_id: uuid (FK → users.id)
name: text
slug: text (UNIQUE)
is_suspended: boolean (default: false)
created_at: timestamptz
updated_at: timestamptz
```

**Notes:**
- Created automatically via `create_account_atomic()` on signup
- One user can own/belong to multiple accounts

---

### 3. account_members
```sql
id: uuid (PK)
account_id: uuid (FK → accounts.id)
user_id: uuid (FK → users.id)
role: enum ('owner', 'admin', 'member')
invited_by: uuid (FK → users.id) | null
created_at: timestamptz
```

**Unique constraint:** (account_id, user_id)

---

### 4. business_profiles ⚠️ PRIMARY TABLE FOR ONBOARDING
```sql
-- Core Identity
id: uuid (PK)
account_id: uuid (FK → accounts.id)
business_name: text NOT NULL
business_slug: text (UNIQUE)
signature_name: text NOT NULL           -- NEW: Moved from users table
website: text | null

-- AI-Generated Content
business_one_liner: text
business_summary: text | null            -- NEW: AI-generated paragraph (editable by user)
ideal_customer_profile: jsonb            -- RENAMED from business_context_pack
operational_metadata: jsonb              -- NEW: Internal app logic data

-- Legacy Fields (deprecated, use ideal_customer_profile instead)
business_niche: text | null              -- DEPRECATED
target_audience: text                    -- DEPRECATED  
industry: text | null                    -- DEPRECATED
business_one_liner: text | null          -- DEPRECATED (merged into business_summary)

-- Context Metadata
context_version: text | null
context_generated_at: timestamptz | null
context_manually_edited: boolean (default: false)
context_updated_at: timestamptz | null

-- Timestamps
created_at: timestamptz
updated_at: timestamptz
deleted_at: timestamptz | null
```

---

### 5. credit_balances
```sql
id: uuid (PK)
account_id: uuid (FK → accounts.id, UNIQUE)
current_balance: integer
last_transaction_at: timestamptz
created_at: timestamptz
updated_at: timestamptz
```

**Notes:**
- Initialized with 25 free credits on account creation
- Created automatically via `create_account_atomic()`

---

### 6. refresh_tokens
```sql
id: uuid (PK)
token: text (UNIQUE)
user_id: uuid (FK → users.id)
account_id: uuid (FK → accounts.id)
expires_at: timestamptz
revoked_at: timestamptz | null
replaced_by_token: text | null
created_at: timestamptz
```

**Notes:**
- Created in custom auth system (Phase 1-2 complete)
- 30-day expiration
- Token rotation on refresh

---

## JSONB STRUCTURE REFERENCE

### ideal_customer_profile (AI-focused data)
```json
{
  // User Inputs (from onboarding)
  "business_description": "We help healthcare providers streamline...",
  "target_audience": "Small business owners in healthcare with 10-50 employees...",
  "industry": "Healthcare",
  "icp_min_followers": 1000,
  "icp_max_followers": 50000,
  "brand_voice": "professional",
  "outreach_goals": "lead-generation",
  
  // AI-Generated Fields (from /v1/generate-business-context)
  "offering": "Healthcare consulting services",
  "icp_industry_niche": "Healthcare",
  "icp_min_engagement_rate": 2.5,
  "icp_content_themes": ["wellness", "medical"],
  "icp_geographic_focus": "US",
  "selling_points": ["Fast ROI", "HIPAA compliant"]
}
```

### operational_metadata (Internal app logic)
```json
{
  "company_size": "1-10",
  "monthly_lead_goal": 50,
  "primary_objective": "lead-generation",
  "challenges": ["time-consuming", "low-quality-leads"],
  "target_company_sizes": ["startup", "smb"],
  "communication_channels": ["email", "instagram"],
  "communication_tone": "professional",
  "team_size": "just-me",
  "campaign_manager": "myself"
}
```

---

## CRITICAL FUNCTIONS

### create_account_atomic(p_user_id, p_email, p_full_name, p_avatar_url)
**Purpose:** Atomic transaction for new user signup

**Returns:**
```json
{
  "user_id": "uuid",
  "account_id": "uuid",
  "is_new_user": true,
  "email": "user@example.com",
  "full_name": "John Doe",
  "onboarding_completed": false,
  "credit_balance": 25,
  "created_at": "2025-01-22T12:00:00Z"
}
```

**What it does:**
1. Inserts/updates user in `users` table
2. Creates account in `accounts` table (if new user)
3. Creates `account_members` entry (role: 'owner')
4. Initializes `credit_balances` with 25 credits
5. Logs credit grant in `credit_ledger`
6. Returns all data needed for JWT payload

**Usage:** Called in OAuth callback handler after Google authentication

---

### generate_slug(input_text) → text
**Purpose:** Generate URL-safe slugs for accounts and business profiles

**Example:**
```sql
SELECT generate_slug('Acme Healthcare Consulting');
-- Returns: 'acme-healthcare-consulting'
```

---

## RLS POLICIES (Row Level Security)

### Onboarding Enforcement
Users cannot access these tables until `onboarding_completed = true`:
- `analyses`
- `leads`
- `business_profiles` (read only)
- `credit_ledger` (read only)

**Exception:** Users CAN write to `business_profiles` during onboarding completion.

---

## ONBOARDING DATA FLOW

### Phase 1: OAuth Callback (Already Complete)
```
Google OAuth → Worker receives code
↓
Worker calls create_account_atomic()
↓
Returns: user_id, account_id, onboarding_completed: false
↓
Worker generates JWT with onboarding_completed: false
↓
Frontend stores tokens
↓
Frontend redirects based on onboarding_completed:
  - false → /onboarding
  - true → /dashboard
```

### Phase 2: Onboarding Completion (To Build - Phase 3)
```
User fills 8-step form
↓
Frontend: POST /api/onboarding/complete with all fields
↓
Worker validates input (Zod schemas)
↓
Worker transforms data:
  - ideal_customer_profile (JSONB)
  - operational_metadata (JSONB)
↓
Worker: INSERT INTO business_profiles (...)
↓
Worker: UPDATE users SET onboarding_completed = true, onboarding_completed_at = now()
↓
Worker: Call AI endpoint /v1/generate-business-context (BLOCKING - 10-15s)
  - Input: business_description, target_audience, industry, operational_metadata
  - Output: business_summary, expanded ideal_customer_profile
↓
Worker: UPDATE business_profiles SET business_summary = ..., ideal_customer_profile = ...
↓
Worker generates NEW JWT with onboarding_completed: true
↓
Worker returns: { success: true, access_token: newJWT, business_profile_id: uuid }
↓
Frontend replaces old token with new token
↓
Frontend redirects to /dashboard
```

---

## ONBOARDING FIELD MAPPING

### Step 1: Personal Identity
| Field | Type | Storage | Validation |
|-------|------|---------|------------|
| `signature-name` | text | `business_profiles.signature_name` | 2-50 chars, required |

### Step 2: Business Basics
| Field | Type | Storage | Validation |
|-------|------|---------|------------|
| `company-name` | text | `business_profiles.business_name` | min 2 chars, required |
| `business-description` | textarea | `ideal_customer_profile.business_description` | 50-500 chars, required |
| `industry` | select | `ideal_customer_profile.industry` | required |
| `industry-other` | text | `ideal_customer_profile.industry` | max 30 chars, required if "Other" |
| `company-size` | radio | `operational_metadata.company_size` | "1-10", "11-50", "51+", required |
| `website` | url | `business_profiles.website` | valid URL, optional |

### Step 3: Goals
| Field | Type | Storage | Validation |
|-------|------|---------|------------|
| `primary-objective` | radio | `operational_metadata.primary_objective` | enum, required |
| `monthly-lead-goal` | number | `operational_metadata.monthly_lead_goal` | 1-10000, required |

### Step 4: Challenges
| Field | Type | Storage | Validation |
|-------|------|---------|------------|
| `challenges` | checkbox[] | `operational_metadata.challenges` | array, optional |

### Step 5: Target Audience
| Field | Type | Storage | Validation |
|-------|------|---------|------------|
| `target-description` | textarea | `ideal_customer_profile.target_audience` | 20-500 chars, required |
| `icp-min-followers` | number | `ideal_customer_profile.icp_min_followers` | min 0, required |
| `icp-max-followers` | number | `ideal_customer_profile.icp_max_followers` | min 0, required |
| `target-size` | checkbox[] | `operational_metadata.target_company_sizes` | array, optional |

### Step 6: Communication
| Field | Type | Storage | Validation |
|-------|------|---------|------------|
| `communication` | checkbox[] | `operational_metadata.communication_channels` | array, required |
| `communication-tone` | radio | `ideal_customer_profile.brand_voice` | enum, required |

### Step 7: Team
| Field | Type | Storage | Validation |
|-------|------|---------|------------|
| `team-size` | radio | `operational_metadata.team_size` | enum, required |
| `campaign-manager` | radio | `operational_metadata.campaign_manager` | enum, required |

### Step 8: Confirmation
No new fields collected - review and submit.

---

## MIGRATION SQL

```sql
-- 1. Add signature_name to business_profiles
ALTER TABLE business_profiles 
ADD COLUMN signature_name TEXT;

-- 2. Create operational_metadata column
ALTER TABLE business_profiles 
ADD COLUMN operational_metadata JSONB DEFAULT '{}'::jsonb;

-- 3. Rename business_context_pack → ideal_customer_profile
ALTER TABLE business_profiles 
RENAME COLUMN business_context_pack TO ideal_customer_profile;

-- 4. Add business_summary column
ALTER TABLE business_profiles 
ADD COLUMN business_summary TEXT;

-- 5. Remove deprecated columns (optional - for cleanup later)
-- ALTER TABLE business_profiles DROP COLUMN business_description;
-- ALTER TABLE business_profiles DROP COLUMN icp_description;

-- 6. Add index for signature_name
CREATE INDEX idx_business_profiles_signature_name 
ON business_profiles(signature_name) 
WHERE deleted_at IS NULL;

-- 7. Migrate existing signature_name data from users
UPDATE business_profiles bp
SET signature_name = u.signature_name
FROM users u
WHERE bp.account_id IN (
  SELECT account_id FROM account_members WHERE user_id = u.id
)
AND u.signature_name IS NOT NULL
AND bp.signature_name IS NULL;

-- 8. Drop signature_name from users (after migration confirmed)
-- ALTER TABLE users DROP COLUMN signature_name;
```

---

## KEY INSIGHTS

1. **Multi-tenancy aware:** Users can belong to multiple accounts; signature_name is business-profile-scoped
2. **JSONB strategy:** Splits AI-focused data (`ideal_customer_profile`) from internal metadata (`operational_metadata`)
3. **AI integration:** Business summary generated via existing `/v1/generate-business-context` endpoint
4. **Blocking onboarding:** AI generation happens during onboarding submit (10-15s with loading screen)
5. **JWT update required:** After onboarding, new JWT issued with `onboarding_completed: true`
6. **RLS enforcement:** Database blocks access to main tables until onboarding complete

---

## VALIDATION ENUMS

```typescript
// Step 2
INDUSTRY = ['Technology', 'Healthcare', 'Finance', 'Real Estate', 'Retail', 
            'Manufacturing', 'Consulting', 'Marketing', 'Education', 'Other']
COMPANY_SIZE = ['1-10', '11-50', '51+']

// Step 3
PRIMARY_OBJECTIVE = ['lead-generation', 'sales-automation', 'market-research', 'customer-retention']

// Step 4
CHALLENGES = ['low-quality-leads', 'time-consuming', 'expensive-tools', 
              'lack-personalization', 'poor-data-quality', 'difficult-scaling']

// Step 5
TARGET_SIZE = ['startup', 'smb', 'enterprise']

// Step 6
COMMUNICATION_CHANNELS = ['email', 'instagram', 'sms']
BRAND_VOICE = ['professional', 'friendly', 'casual']

// Step 7
TEAM_SIZE = ['just-me', 'small-team', 'large-team']
CAMPAIGN_MANAGER = ['myself', 'sales-team', 'marketing-team', 'mixed-team']
```

---

## NEXT PHASE: WORKER ENDPOINT IMPLEMENTATION

**File to create:** `src/features/onboarding/onboarding.handler.ts`

**Endpoint:** `POST /api/onboarding/complete`

**Required logic:**
1. Validate all 14 fields with Zod
2. Transform to `ideal_customer_profile` + `operational_metadata`
3. Generate business_slug from business_name
4. INSERT business_profile
5. UPDATE users.onboarding_completed
6. Call AI endpoint (blocking)
7. UPDATE business_profile with AI results
8. Generate new JWT
9. Return success + new token

---

**END OF DOCUMENT**
