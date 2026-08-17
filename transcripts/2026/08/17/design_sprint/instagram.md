# Design Sprint Transcript (front half, timeboxed)
**Date:** 2026-08-17
**Start Time:** 12:03:14 · **End Time:** 12:23:20
**Problem:** Instagram — photo upload with captions, follow graph, reverse-chron feed, likes and comments
**Difficulty:** 3/5 — one real scale break (100:1 read skew against a 200ms p99 feed SLA) and one genuine trade-off (serve originals vs renditions)
**Front-half readiness: 2/5**
**Front half complete inside 17:00: no** — API landed at 20:06. At the buzzer there was no like-write or comment-write endpoint, no delete or unfollow endpoint anywhere, no error/status semantics on any endpoint, no named fields inside `photoMetadata`, and no `nextCursor` on the likes endpoint.

> Sitting stopped after problem 1 of 2 by request (problem 2, online auction 4/5, served at 12:23:20 and not attempted).

| Phase | Pace target | Landed at | ± vs target | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 8:00 | 9:12 | +1:12 | 3 (FR block, 1 scale question, NFR block) | 3/5 |
| Core entities | 12:00 | 11:54 | −0:06 | 1 | 2/5 |
| API design | 17:00 | 20:06 | +3:06 | 1 | 2/5 |

**Budget allocation:** 0:00–3:00 FRs + out of scope · 3:00–4:03 scale question · 4:03–9:12 NFRs (5:09) · 9:12–11:54 entities (2:42, first phase in the record to land under target) · 11:54–20:06 API (8:12, 40% of the box, still overran by 3:06). Requirements consumed 54% of the 17-minute box; the API phase paid for it and still shipped no error semantics.

**First-pass completeness:** One message per phase, zero back-fill, zero revisions, and the single clarifying question was correctly batched into one ask (received the whole givens block in one reply). Nothing message two had to rescue — the pace failure is composition throughput, not incomplete first passes. Separately and distinctly: the API first pass was missing whole *categories* (writes for FR4, all deletes, all error semantics) that were never back-filled — a scaffold with missing rows, not slow thinking. Different fix.

**Plausibility check:** Not performed. "Need hot and cold tier storage" is a consequence of 200 TB/day, not a check on it. Never stated 73 PB/year, never derived the 200 GB/s egress that 100k reads/s × 2MB implies, never cost-checked "kept forever", never questioned his own 5-photos-per-feed-open assumption (a real scroll is 10–20). Requirements capped at 3 on this basis. What it would have caught: that originals cannot be served on the read path, which is the entire reason a `Rendition` entity and a CDN exist in this design.

**Arithmetic:** fully self-consistent for the first time in the record. 100M × 1/day ÷ 10⁵ = 1k uploads/s → 2 GB/s → 200 TB/day; 100M × 20 opens ÷ 10⁵ = 20k opens/s × 5 = 100k reads/s; 100k:1k = 100:1 matches both QPS figures. Against 24 sessions of BoE slips this is the cleanest traffic model produced to date.

## What he produced (verbatim, as it stood at 17:00)

### Requirements
```
FRs ->
1. Users upload photos with captions.
2. Users can follow other users
3. Users can see feed of photos from people they follow.
4. Photos can have likes and comments.

Out of scope ->
1. Analytics.
2. Feed recommendation

NFRs ->
1. Highly available for photo upload (99.99 ~ 52 mins downtime/year)
2. Eventual consistency acceptable for uploaded photos, and also likes and comments.
3. Low latency feed (p99 < 200ms)
4. Durable photo uploads.
5. 100M DAU with 1 photo upload/user/day and 2MB photo -> 1k photos uploads/s -> 2 GB/s ingress.
   (200TB/day - Need hot and cold tier storage )
   With 20 feed opens/user/day and each feed having 5 photos -> 100k photo reads/s avg (peak 3x).
   Highly read skewed system. Read:Wrtie (100:1)
```
Clarifying questions asked: one — "what scale does the system need to be designed for?" (at 3:00, answered 4:03).

### Core entities
```
1. User (id, name, email)
2. PhotoMediaBytes stored in s3
3. Post (id, userId, text, photoS3Url)
4. FollowRelationship(followingUserId, followedUserId)
5. Likes (postId, userId) (UNIQUE constraint on postId+userId)
6. Comment (postId, text, userId, createdAt)
```

### API design
```
NOTE: Identity extracted from auth header

1. POST /posts
Request: {text, photoMetadata}
Response: Post(id, photS3Url(s3 presigned URL for client to upload to s3),status:"UPLOADING")
Idempotecy header

2. PUT /posts/:id/complete
Response: 2xx status indicating Post status updated to UPLOADED.

3. POST /follow/:userId
Response: 2xx status indicating user followed.

4. GET /feed?cursor={}&limit={}
Response: Post(id, photoS3lUrl, text, likeCount, commentCount)[], nextCursor:"postId + postTime"
Cursor based pagination to allow for dynamically inserting new photos

5. GET /posts/:id/likes?cursor={}&limit={}
Response: Like(postId, userId)[]

6. GET /posts/:id/comments?cursor={}&limit={}
Response: Comment(id, postId, userId, text, createdAt)[], nextCursor:
```

## What was still missing at 17:00

**Requirements:** no plausibility check on any number · 73 PB/year never stated · 200 GB/s egress never derived · no consistency posture for the follow graph (the follow→feed staleness question) · out-of-scope list omits stories, DMs, search, notifications · durability stated as a bare word with no target.

**Entities:** no `FeedEntry`/materialised timeline — the object his own 200ms p99 at 20k opens/s requires · no `Rendition`/thumbnail entity despite serving 2MB originals on the read path · **`likeCount` and `commentCount` are returned by the feed API but exist in no entity** · **the feed cursor is "postId + postTime" but `Post` has no timestamp field** · no partition or primary key stated on any entity · `PhotoMediaBytes stored in s3` is a storage decision rather than an entity.

**API:** no `POST /posts/:id/likes` and no `POST /posts/:id/comments` — FR4 is in scope with no write path, and these are the two highest-QPS writes after upload · no deletes at all (post, comment, unlike, **unfollow**) · no error/status semantics anywhere, including no resolution for a `/complete` called twice or never (the orphaned `UPLOADING` post is the native failure mode of his own upload design) · `photoMetadata` names no fields · `GET /posts/:id/likes` has no `nextCursor` and returns bare `userId`s no client can render · `nextCursor:` on comments left literally blank · no `GET /posts/:id` and no `GET /users/:id/posts` (profile grid) · no page-size caps · `photS3Url` typo carried from entities.

## Where the time went
NFRs took 5:09 and the API took 8:12 — together 79% of the box. Nothing was re-derived and nothing was revised; the scaffold was being filled in, not rebuilt. Entities landed under target for the first time. The overrun is pure throughput on the two long-form phases, compounded on the API by writing endpoints in flow order (upload → complete → follow → feed → likes → comments) rather than deriving them from the FR list, which is why three FRs ended up with reads and no writes.

Genuinely senior work to preserve: the presigned-URL two-phase upload with a `status` enum the client branches on, volunteered unprompted, which keeps 2 GB/s of bytes off the API tier; cursor pagination with a stated justification on the feed and comments; an idempotency header on the upload; identity-from-auth stated once instead of repeated per endpoint; and the `UNIQUE(postId,userId)` constraint on `Like`.

## Ideal front half (writable in the same 17 minutes)

### Requirements (target 8:00)
**FRs:** 1. Upload photo + caption. 2. Follow/unfollow. 3. Reverse-chron feed of followees' posts. 4. Like/unlike and comment. 5. Profile grid.
**Out of scope:** stories, DMs, reels, search/explore, recommendation ranking, notifications, analytics, moderation.
**NFRs:**
- **Availability:** feed reads 99.99% (52 min/yr); uploads 99.9% — a failed upload is client-retryable, a dead feed is the product being down.
- **Consistency per subsystem:** feed eventual (seconds of staleness fine) · follow graph read-your-own-writes for the acting user · likes/comments eventual counts but exactly-once per (post,user) via the unique key · a post is invisible until `complete`.
- **Latency:** feed p50 < 100ms, p99 < 200ms (metadata only — bytes from CDN); upload ack p99 < 300ms since bytes bypass the API.
- **Traffic:** 100M × 1 = 10⁸/10⁵ = **1k writes/s** (peak 3k). 100M × 20 = 2×10⁹/10⁵ = **20k opens/s** × 5 = **100k reads/s** (peak 300k). **R:W ≈ 100:1.**
- **Storage:** 2 GB/s = 200 TB/day = **73 PB/yr** of originals.
- **Bandwidth:** 100k × 2MB = **200 GB/s egress** if originals are served — 100× ingress.
- **Plausibility check, out loud:** *"73 PB/yr kept forever is ~$1.5M/month of hot object storage growing linearly, so originals tier to cold after ~30 days and reads serve a ~200KB rendition — dropping egress from 200 GB/s to ~20 GB/s, which is a CDN's job, not the origin's. And 5 posts per feed open is conservative; a real scroll is 10–20, so treat 100k reads/s as a floor."*
- **Durability:** photos 11 nines; metadata replicated, no acknowledged write lost.

> **What this buys:** his arithmetic was already correct — this adds the two figures he stopped one multiplication short of (73 PB/yr, 200 GB/s) plus the one sentence that tests them, and those two numbers are what force renditions, tiering and the CDN, i.e. most of the HLD. Splitting consistency per subsystem also surfaces the follow→feed staleness question before the deep dive.

### Core entities (target 12:00)
```
User        (user_id PK, handle UNIQUE, name, follower_count↓, post_count↓)
Post        (post_id PK [snowflake: time-sortable], author_id, caption, media_id,
             status ENUM{UPLOADING,READY,FAILED}, created_at, like_count↓, comment_count↓)
Media       (media_id PK, owner_post_id, original_key, bytes, content_type)
Rendition   (media_id + variant PK — variant ∈ {thumb,feed,full}, cdn_key, bytes)
Follow      (follower_id + followee_id PK/UNIQUE, created_at)
Like        (post_id + user_id PK/UNIQUE, created_at)      ← uniqueness = idempotency
Comment     (comment_id PK, post_id, author_id, text, created_at)
FeedEntry   (user_id + post_id PK, author_id, created_at)  ← partition by user_id
```
`↓` = denormalised counter maintained asynchronously; the API returns these, so they must exist as state.

**Entities that only appear under load:** **`FeedEntry`** — a read-time fan-in across 500 followees cannot hold a 200ms p99 at 20k opens/s, so the timeline is materialised per reader on write. **`Rendition`** — forced by the 200 GB/s egress figure; the feed never reads a 2MB original. Neither exists in a small-scale version of this product.

> **What this buys:** it closes both places where his API returned state his model lacked (`likeCount`/`commentCount` → explicit denormalised counters; `postTime` in the cursor → `created_at` plus a time-sortable `post_id` so the cursor is one opaque field), and it names the timeline object his own 200ms p99 requires.

### API design (target 17:00)
Identity from the auth header throughout; `Idempotency-Key` on every unsafe method.
```
POST   /v1/posts                        {caption, content_type, bytes, width, height}
       → 201 {post_id, status:"UPLOADING", upload_url, upload_expires_at}
PUT    /v1/posts/{post_id}/complete     {etag}
       → 200 {post_id, status:"READY"} | 409 already READY | 410 url expired
DELETE /v1/posts/{post_id}              → 204 | 403 not author | 404
GET    /v1/posts/{post_id}              → 200 {post_id, author, caption,
                                          renditions{thumb,feed}, like_count,
                                          comment_count, viewer_liked, created_at}
GET    /v1/users/{user_id}/posts?cursor&limit≤50  → {items[], next_cursor}
GET    /v1/feed?cursor&limit≤20                   → {items[PostSummary], next_cursor}
       cursor = opaque base64(post_id); post_id is time-sortable so the cursor is stable
       while new posts arrive at the head — offset would skip and repeat rows.
PUT    /v1/posts/{post_id}/like         → 204   (PUT not POST: idempotent by the
DELETE /v1/posts/{post_id}/like         → 204    (post_id,user_id) unique key)
POST   /v1/posts/{post_id}/comments     {text} → 201 {comment_id, created_at}
DELETE /v1/comments/{comment_id}        → 204 | 403
GET    /v1/posts/{post_id}/comments?cursor&limit≤50 → {items[], next_cursor}
PUT    /v1/users/{user_id}/follow       → 204
DELETE /v1/users/{user_id}/follow       → 204
GET    /v1/users/{user_id}/followers?cursor&limit≤100 → {items[UserSummary], next_cursor}
```
**Errors:** 400 validation · 401 · 403 not owner · 404 · 409 state conflict (double-complete) · 413 media too large · 429 + `Retry-After` (like/comment abuse) · 503 retry-safe on all idempotent methods. Orphaned `UPLOADING` posts reaped by TTL — so the failure mode of the two-phase upload has a named owner.

> **What this buys:** one endpoint per FR verb, which supplies the like/comment/unfollow/delete writes he never wrote; `viewer_liked`, a field the client needs and cannot compute; a size cap on every list; named fields in the one request body he did have; and error semantics that include the double-`complete` case his own design creates. Using `PUT` for like and follow buys idempotency without a key.

## Feedback given
- Arithmetic fully self-consistent for the first time in the record — 1k uploads/s → 2 GB/s → 200 TB/day, 20k opens/s → 100k reads/s, 100:1 ratio matching both figures. Percentile on the latency NFR, read:write and storage growth both stated. Three previously-dropped items delivered.
- Requirements capped at 3 because no number was sanity-checked. Stopped one multiplication short of the figure that decides the design: 100k reads/s × 2MB = 200 GB/s egress.
- Entities returned `likeCount`/`commentCount` and a `postTime` cursor against a model containing none of them — the same API-promises-state-the-model-lacks defect as the previous day's `Booking.user`. Whatever the response returns, check it exists in an entity.
- The load-bearing entity (`FeedEntry`) was absent while its SLA was committed to.
- FR4 shipped with reads and no writes; nothing in the API can create a like or a comment, nothing can delete or unfollow, and no endpoint has error semantics.
- Real credit: presigned two-phase upload with a status enum, cursor pagination with a justification, idempotency header, and the `UNIQUE(postId,userId)` constraint — all unprompted.
- The one habit to change: **start the API phase by writing one line per FR listing its create/read/delete, then fill in paths and shapes.** Derive the endpoint list from the FRs, not from the upload flow.
