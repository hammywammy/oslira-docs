# 📊 Instagram Profile Data Schema

> Complete field reference for Instagram profile scraping data structure.

---

&nbsp;

## 🧑‍💼 Profile-Level Fields

Core fields describing the main profile entity.

&nbsp;

### Identifiers

| Field | Description |
|:------|:------------|
| `id` | Unique identifier for the profile entity |
| `fbid` | Facebook ID associated with the profile |
| `username` | The handle associated with the profile |
| `url` | Direct URL to the Instagram profile |
| `fullName` | User's display name |

&nbsp;

---

&nbsp;

### Profile Text & Bio

| Field | Description |
|:------|:------------|
| `biography` | Text describing the user |

&nbsp;

---

&nbsp;

### External Links

| Field | Description |
|:------|:------------|
| `externalUrl` | Primary external link displayed on the profile |
| `externalUrlShimmed` | Redirect/shimmed tracking URL version |
| `externalUrls` | Array of structured external links |

**`externalUrls[]` Object Structure:**

| Field | Description |
|:------|:------------|
| `title` | Display title of the link |
| `url` | Clean destination URL |
| `lynx_url` | Instagram's tracking/redirect URL |
| `link_type` | Type classification (e.g., `external`) |

&nbsp;

---

&nbsp;

### Counts & Stats

| Field | Description |
|:------|:------------|
| `followersCount` | Number of followers |
| `followsCount` | Number of accounts followed |
| `postsCount` | Count of posts on the profile |
| `highlightReelCount` | Number of story highlights |
| `igtvVideoCount` | Count of IGTV-format videos |

&nbsp;

---

&nbsp;

### Account Properties & Flags

| Field | Description |
|:------|:------------|
| `private` | Whether the account is private |
| `verified` | Whether the account is verified |
| `isBusinessAccount` | Whether the profile is a business account |
| `businessCategoryName` | Category associated with a business profile |
| `hasChannel` | Whether the user has a broadcast channel |
| `joinedRecently` | Flag indicating whether the account is new |

&nbsp;

---

&nbsp;

### Images

| Field | Description |
|:------|:------------|
| `profilePicUrl` | Standard-resolution profile picture |
| `profilePicUrlHD` | High-definition profile picture |

&nbsp;

---

&nbsp;

### Source Metadata

| Field | Description |
|:------|:------------|
| `inputUrl` | URL from which the data was scraped |

&nbsp;

---

&nbsp;

## 👥 Related Profiles

Suggested accounts / potential leads returned in `relatedProfiles[]`.

&nbsp;

| Field | Description |
|:------|:------------|
| `id` | Unique ID for the related profile |
| `username` | Handle of the related profile |
| `full_name` | Display name |
| `is_verified` | Verification status |
| `is_private` | Privacy status |
| `profile_pic_url` | URL of the related profile's picture |

&nbsp;

---

&nbsp;

## 📸 Latest Posts

Each object inside `latestPosts[]` contains the following fields.

&nbsp;

### Identifiers

| Field | Description |
|:------|:------------|
| `id` | Unique ID of the post |
| `shortCode` | Short permalink identifier |
| `url` | Public URL linking to the post |
| `ownerId` | ID of the content owner |
| `ownerUsername` | Username of content owner |

&nbsp;

---

&nbsp;

### Content Metadata

| Field | Description |
|:------|:------------|
| `type` | Type of post (`Video`, `Image`, `Sidecar`) |
| `productType` | Post classification (`clips`, `feed`, etc.) |
| `caption` | Text caption of the post |
| `hashtags` | List of hashtags extracted from caption |
| `mentions` | List of mentioned users |
| `alt` | Accessibility alt text for the media |

&nbsp;

---

&nbsp;

### Media URLs & Assets

| Field | Description |
|:------|:------------|
| `displayUrl` | Thumbnail or displayed media URL |
| `videoUrl` | Direct URL to video file (if video content) |
| `images` | Array of image URLs (for carousels) |

&nbsp;

---

&nbsp;

### Dimensions

| Field | Description |
|:------|:------------|
| `dimensionsHeight` | Height of media in pixels |
| `dimensionsWidth` | Width of media in pixels |

&nbsp;

---

&nbsp;

### Engagement Stats

| Field | Description |
|:------|:------------|
| `likesCount` | Number of likes |
| `commentsCount` | Number of comments |
| `videoViewCount` | Number of video views |

&nbsp;

---

&nbsp;

### Timing

| Field | Description |
|:------|:------------|
| `timestamp` | ISO 8601 date/time when the post was created |

&nbsp;

---

&nbsp;

### Location

| Field | Description |
|:------|:------------|
| `locationName` | Human-readable location name |
| `locationId` | Instagram's location ID |

&nbsp;

---

&nbsp;

### Moderation

| Field | Description |
|:------|:------------|
| `isCommentsDisabled` | Whether commenting is turned off |

&nbsp;

---

&nbsp;

### Tagged Users

Array `taggedUsers[]` containing:

| Field | Description |
|:------|:------------|
| `id` | User ID of the tagged person |
| `username` | Username of the tagged person |
| `full_name` | Display name of the tagged person |
| `is_verified` | Verification status |
| `profile_pic_url` | Profile picture URL |

&nbsp;

---

&nbsp;

### Nested / Carousel Posts

For `Sidecar` (carousel) posts, `childPosts[]` contains nested post objects with the same structure as parent posts.

&nbsp;

---

&nbsp;

## 📺 IGTV Videos

Array `latestIgtvVideos[]` — uses the same field structure as `latestPosts[]`.

&nbsp;


Now I can map the exact schema fields to the 5-tier framework. Let me cross-reference your actual data structure with the applications:

---

## **TIER 1: CRITICAL** (Must-use for every lead score)

| Field(s) | Derived Metric | Application |
|:---------|:---------------|:------------|
| `latestPosts[].likesCount` + `latestPosts[].commentsCount` ÷ `followersCount` | **Engagement Rate** | Core authenticity signal — below 0.3% = fake/dead audience |
| `followersCount` ÷ `followsCount` | **Authority Ratio** | Below 1:1 = follow-for-follow desperation, above 5:1 = genuine influence |
| `postsCount` + `latestPosts[].timestamp` spread | **Posting Consistency** | Calculate posts/month average — proves dedicated marketing resource |
| `isBusinessAccount` + `businessCategoryName` | **Commercial Intent** | Non-business = hobbyist; specific category = positioning clarity |
| `biography` | **Bio Quality** | Presence of value prop, CTA, credentials = professional marketing ops |
| `externalUrl` or `externalUrls[]` | **Conversion Focus** | No link = no funnel; link-in-bio tool = automation maturity |
| `latestPosts[].commentsCount` ÷ `latestPosts[].likesCount` | **Engagement Depth** | 1:10-1:20 = active community; 1:50+ = passive scrollers |

---

## **TIER 2: VERY VALUABLE** (Reveals operational maturity)

| Field(s) | Derived Metric | Application |
|:---------|:---------------|:------------|
| `latestPosts[].type` distribution | **Format Diversity** | All `Image` = basic; mix of `Video`, `Sidecar`, `Image` = sophisticated |
| `latestPosts[].productType` | **Content Investment** | `clips` (Reels) presence = platform adaptation, higher production |
| `latestPosts[].caption` | **Caption Quality** | Length, structure, CTAs, hooks = copywriting resources |
| `externalUrls[].url` domain analysis | **Tech Stack Signals** | Calendly = booking system; Linktree = multi-link; custom domain = advanced |
| `latestPosts[].timestamp` | **Posting Times** | Consistent hours = scheduling tools; business hours = professional ops |
| `latestPosts[].hashtags` | **Discovery Strategy** | 3-10 niche tags = strategic; 30 generic = spam approach |
| `verified` | **Platform Authority** | Verified = established brand, media presence, trust signal |
| `highlightReelCount` | **Content Curation** | 5+ organized highlights = evergreen content strategy |
| `latestPosts[].videoViewCount` ÷ `followersCount` | **Video Performance** | View rate reveals content reach beyond static engagement |

---

## **TIER 3: VALUABLE** (Contextual enrichment)

| Field(s) | Derived Metric | Application |
|:---------|:---------------|:------------|
| `followersCount` trend (if tracked over time) | **Growth Trajectory** | Requires historical data — rising = momentum, flat = plateau |
| `latestPosts[].taggedUsers[]` | **Partnership Signals** | Verified tagged users = brand partnerships; recurring tags = client relationships |
| `latestPosts[].mentions` | **Network Activity** | Outbound mentions reveal collaborations, vendors, testimonials |
| `latestPosts[].locationName` | **Geographic Scope** | Single city = local; multiple = regional/national; none = digital-first |
| `hasChannel` | **Broadcast Capability** | Has channel = community investment, direct communication path |
| `latestPosts[].isCommentsDisabled` | **Engagement Stance** | Disabled = avoiding feedback (red flag) or spam protection |
| `relatedProfiles[]` | **Lookalike Prospecting** | Algorithm-suggested similar accounts for list building |
| `latestPosts[].alt` | **Accessibility Investment** | Custom alt text = attention to detail, professional content ops |

---

## **TIER 4: CONDITIONAL VALUE** (Only useful cross-referenced)

| Field(s) | Potential Use | Limitation |
|:---------|:--------------|:-----------|
| `relatedProfiles[].username` | Lookalike list building | Requires manual validation — algorithm suggestions aren't qualified leads |
| `latestPosts[].caption` language analysis | Customer-focus vs product-focus | Needs NLP; single posts misleading without pattern |
| `businessCategoryName` + `followersCount` combo | Revenue range estimation | Wide confidence interval — directional only |
| `latestPosts[].childPosts[]` (carousels) | Content depth analysis | Carousel presence = effort, but slide count alone isn't signal |
| `latestPosts[].timestamp` gaps | Activity patterns | Could mean vacation, pivot, or decline — ambiguous without context |
| `latestIgtvVideos[]` | Long-form investment | IGTV deprecated — legacy content, low current relevance |

---

## **TIER 5: BACKGROUND/EXCESS** (Store but don't score)

| Field(s) | Why Excluded |
|:---------|:-------------|
| `id`, `fbid`, `ownerId` | Technical identifiers — no business signal |
| `followersCount` alone | Vanity metric without engagement context |
| `likesCount` alone | Meaningless without ratio to followers |
| `url`, `inputUrl` | Reference data only |
| `profilePicUrl`, `profilePicUrlHD` | Display purposes — no qualification value |
| `externalUrlShimmed`, `lynx_url` | Instagram internal tracking — no user signal |
| `dimensionsHeight`, `dimensionsWidth` | Platform formatting metadata |
| `shortCode`, `latestPosts[].id` | Permalink identifiers only |
| `joinedRecently` | Weak signal — new accounts can be established businesses rebranding |
| `igtvVideoCount` | Deprecated feature count |
| `private` | If true, you can't analyze anyway — gate check, not scoring |
| `latestPosts[].displayUrl`, `videoUrl`, `images` | Media asset URLs — display only |
| `relatedProfiles[].profile_pic_url` | Display asset |
| `taggedUsers[].profile_pic_url` | Display asset |

---

## Summary: Field Count by Tier

| Tier | Field Groups | Purpose |
|:-----|:-------------|:--------|
| **Critical** | 7 derived metrics | Every lead, always |
| **Very Valuable** | 9 derived metrics | Deep qualification |
| **Valuable** | 8 derived metrics | Contextual refinement |
| **Conditional** | 6 field uses | Cross-reference only |
| **Background** | 15+ fields | Store, don't process |

---

**Bottom line**: Your schema has ~40 fields. Only **~24 need active processing** (Tiers 1-3). The rest are either display assets, technical IDs, or require external validation to be meaningful.

Want me to draft the AI prompt structure that feeds only Tier 1-3 data to your analysis models?



SECOND OPINION: 

Got it — here are **practical, plug-and-use formulas** for each of the 5 categories.
These are not explanations — they’re **formulas, scoring logic, boolean checks, and combo rules** you can actually use in code or workflows.

---

# 🟥 **1. CRITICAL — High-Impact Scoring Formulas**

### **(1) Business Relevance Score**

```
BusinessRelevance = 
    (businessCategoryName ? 40 : 0)
  + (isBusinessAccount ? 30 : 0)
  + (verified ? 20 : 0)
  + (externalUrl exists ? 10 : 0)
```

### **(2) Authority Score**

```
AuthorityScore =
    (followersCount * 0.003) 
  + (verified ? 25 : 0)
  + (postsCount * 0.001)
```

### **(3) Intent Strength via Bio**

```
IntentScore =
    (bio contains keywords ? 40 : 0)
  + (bio contains service/offer verbs ? 20 : 0)
  + (bio length > 80 ? 10 : 0)
```

### **(4) Contact Opportunity Flag**

```
ContactReady = externalUrl OR "Email" in bio OR "DM" in bio
```

---

# 🟧 **2. VERY VALUABLE — Behavioral & Engagement Formulas**

### **(5) Engagement Rate**

```
EngagementRate = (likesCount + commentsCount) / followersCount
```

### **(6) Content Activity Score**

```
ContentActivity = 
    (postsCount * 0.01)
  + (highlightReelCount * 0.5)
  + (igtvVideoCount * 0.8)
```

### **(7) Topic Match Strength**

```
TopicMatch =
    (hashtags matching targetKeywords * 5)
  + (mentions of relevant accounts * 8)
```

### **(8) Social Connectivity Score**

```
NetworkStrength =
    (taggedUsers count * 4)
  + (relatedProfiles count * 2)
```

---

# 🟨 **3. VALUABLE — Contextual Supporting Formulas**

### **(9) Brand vs Personal Indicator**

```
BrandLikely =
    (profilePic contains logo ? 20 : 0)
  + (followsCount < followersCount * 0.5 ? 10 : 0)
  + (username contains business keyword ? 10 : 0)
```

### **(10) Content Style Weight**

```
MediaPreference = 
    (Reels posts / total posts) * 10 
  + (Image posts / total posts) * 5
```

### **(11) Caption Insight Score**

```
CaptionInsight = 
    (CTA keywords count * 10)
  + (product/service terms count * 5)
  + (caption length > 100 ? 5 : 0)
```

---

# 🟩 **4. COULD BE GOOD IF APPLIED RIGHT — Advanced Inference**

### **(12) New Account Risk Flag**

```
NewRisk = (joinedRecently ? 1 : 0)
```

### **(13) Professionalism Score (Media Resolution)**

```
MediaQuality = (avg(resolution width * height) / 1,000,000)
```

### **(14) Channel Engagement Potential**

```
ChannelPotential = hasChannel ? 1 : 0
```

### **(15) Geo Strength (if locationId mapped)**

```
GeoStrength = (locationName exists ? 10 : 0)
```

---

# ⚪ **5. BACKGROUND INFO — Low Value Checks**

### **(16) Accessibility Check**

```
ProfileAccessible = NOT private AND NOT is_private
```

### **(17) Comments Availability**

```
CanMeasureEngagement = NOT isCommentsDisabled
```

### **(18) ID Validity Check**

```
ValidProfile = id exists AND fbid exists
```

