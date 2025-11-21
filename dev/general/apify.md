## PROFILE-LEVEL FIELDS

These fields describe the main profile, its metadata, its counts, and its external links.

Identifiers

id
Unique identifier for the profile entity.
Aliases: fbid, ownerId

username
The handle associated with the profile.

fullName
User’s display name.
Aliases: full_name

Profile Text & Bio

biography
Text describing the user.

External Links

externalUrls
Array of structured external links (title, type, URL metadata).

externalUrl
Primary external link displayed on the profile.

externalUrlShimmed
Redirect/shimmed tracking URL version.

Counts & Stats

followersCount
Number of followers.

followsCount
Number of accounts followed.

highlightReelCount
Number of story highlights.

postsCount
Count of posts on the profile.

igtvVideoCount
Count of IGTV-format videos.

Account Properties & Flags

private
Whether the account is private.
Alias: is_private

verified
Whether the account is verified.
Alias: is_verified

isBusinessAccount
Whether the profile is a business account.

businessCategoryName
Category associated with a business profile.

hasChannel
Whether the user has a broadcast channel.

joinedRecently
Flag indicating whether the account is new.

Images

profilePicUrl
Standard-resolution profile picture.
Alias: profile_pic_url

profilePicUrlHD
High-definition profile picture.

Source Metadata

inputUrl
URL from which the data was scraped.

## RELATED PROFILES (LEADS / SUGGESTED ACCOUNTS)

Each object inside relatedProfiles contains:

Identifiers

id
Unique ID for the related profile.

username
Handle of the related profile.

fullName
Display name.
Alias: full_name

Profile Properties

is_verified
Verification status.
Alias: verified

is_private
Privacy status.
Alias: private

Media

profile_pic_url
URL of the related profile’s picture.
Alias: profilePicUrl

## LATEST POSTS (POST-LEVEL FIELDS)

Each object inside latestPosts includes the following fields:

Identifiers

id
Unique ID of the post.

shortCode
Short permalink identifier.

Content Metadata

type
Type of post (video, image, clip, carousel, etc.).

caption
Text caption of the post.

hashtags
List of hashtags extracted from caption.

mentions
List of mentioned users.

URLs & Media

url
Public URL linking to the post.

displayUrl
Thumbnail or displayed media URL.

videoUrl
Direct URL to video file (if video content).

images
Array of image objects or variants.

alt
Accessibility alt text for the media.

Dimensions

dimensionsHeight
Height of media in pixels.

dimensionsWidth
Width of media in pixels.

Engagement Stats

likesCount
Number of likes.

commentsCount
Number of comments.

videoViewCount
Number of video views.

Timing

timestamp
Date/time when the post was created.

Ownership

ownerUsername
Username of content owner.

ownerId
ID of the content owner.
(Merged under id alias group but kept for readability at post-level)

Structural

childPosts
Carousel or nested media items.

Additional Metadata

productType
Post classification (reels, clips, feed, etc.).

isCommentsDisabled
Whether commenting is turned off.
