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
