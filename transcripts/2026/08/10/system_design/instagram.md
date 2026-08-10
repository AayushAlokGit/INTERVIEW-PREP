# System Design Round Transcript
**Date:** 2026-08-10
**Start Time:** 10:33:47 · **End Time:** 11:51:49 · **Duration:** 78 min
**Problem:** Instagram (photo & video sharing)
**Difficulty:** Medium
**Dominant pattern:** large blobs + scaling reads (with fan-out / scaling writes on the feed path)
**Performance Rating:** 3/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->

**Would it have fit a real 45-min round?** No — cut off mid-HLD. The entire deep dive (celebrity fan-out, merged cursor, sizing, operability) would never have been reached.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 19.7 min | +11.7 | No |
| Core entities | 12 min | 22.5 min | +10.5 | No |
| API design | 17 min | 36.0 min | +19.0 | No |
| High-level design | 27 min | 49.4 min | +22.4 | No |
| Deep dive | 40 min | 73.7 min | +33.7 | No |
| Wrap-up | 45 min | 78.0 min | +33.0 | No |
| **Total** | **45 min** | **78 min** | **+33** | **No** |

---

## Conversation Log

**Interviewer:** Design Instagram — a photo and video sharing service. Users upload photos/videos with captions, follow other users, and see a home feed of posts from people they follow. Posts can be liked and commented on. There's also a per-user profile grid of their own posts. Difficulty: Medium. Reference timeline stated (measured, not enforced). Start with requirements.

**Aayush:** In the home feed do the posts need to be arranged in chronological order? And do we need to support recommendation for posts for the home feed?

**Interviewer:** Chronological: yes, reverse-chronological from people you follow. Recommendations: no — no suggested posts from non-followed accounts, no ML ranking. Follow-graph only.

**Aayush:**
FRs:
1. Users can upload photos and videos. They can add captions to their uploads.
2. Users can follow other users.
3. Users can view their home feed of chronologically sorted posts from people they follow.
4. Posts can be liked and commented on.
5. For a user, show a dedicated profile grid for their own posts.

Out of scope:
1. Analytics on posts
2. Recommendation and ranking of posts

**Interviewer:** FRs are clear, and I'll take the explicit out-of-scope list. Now NFRs — with numbers. Scale (users, DAU, posts/day, feed reads/day), read/write ratio, latency targets per operation, and consistency vs availability for each part.

**Aayush:**
NFRs:
1. Highly available for post upload (99.9% ~ 8.6 hrs downtime/year)
2. Low latency feed gen (p99 < 200ms)
3. Eventual consistency acceptable for posts, likes and comments.
4. Durable storage for photos and videos.
5. 100M DAU, each user posting 2 posts a day → 2k posts/s avg, peak 5×. Read/write ratio 10:1 → 20k post reads/s. Each post ~1MB → 2GB/s data.
Need help for estimates of like and comment traffic scale.

**Interviewer:** Take a swing at likes/comments yourself first. Also: (1) sanity-check the 10:1 read/write against your own FRs — 100M DAU × feed loads × posts per load; (2) 1MB/post at 2k/s — what does that mean a year out, and does it change a storage decision? (3) You said eventual consistency is fine for posts, likes and comments — is there any part where that's *not* acceptable?

**Aayush:**
1. 100M DAU, each posting 2 posts/day → 2k posts/s avg, peak 5×. Each user loads feed 5 times/day, each feed has 10 posts → 50k reads/s. Each post ~1MB → 2GB/s.
2. It means we can't have storage in a DB and we need object storage like S3, and we can't retain the data for very long else storage cost blows up. Retention of 1 year ≈ 60 PB.
3. For posts, read-your-own-writes consistency would be needed.

**Interviewer:** 50k post-reads/s reconciles. Two follow-ups: (1) retention — product says posts are permanent, you can't delete after a year. What's your alternative for keeping 60PB/year affordable? (2) Likes/comments volume — still owe me a number.

**Aayush:** 2. Assuming each user likes 20 posts/day and comments on 5 posts/day → 20k likes/s and 5k comments/s.

**Interviewer:** So likes are ~10× your post write rate. Still owe me #1: posts are permanent, how do you keep 60PB/year affordable?

**Aayush:** Offload all storage to S3 which is a cheaper alternative for storing huge scale data.

**Interviewer:** You already said object storage. Sharper: is all 60PB accessed equally? A post from three years ago vs an hour ago — same read frequency? What does that let you do with S3 specifically?

**Aayush:** Hot and cold tiering.

*(Requirements phase complete — 19.7 min elapsed. Requirements rendered to canvas.)*

**Interviewer:** Now core entities. What are the objects, and what fields matter on each?

**Aayush:**
1. User (id, name, email)
2. Post (id, userId, text, mediaS3Url, createdAt)
3. Like (userId, postId) → unique constraint on userId + postId (a user can only like a post once)
4. Comment (id, userId, postId, text, createdAt)
5. FollowRelationship (followingUserId, followedUserId)

**Interviewer:** When I render a feed of 10 posts, each shows a like count and a comment count. Nothing in your entity list holds those. Where do they come from, and does that change any entities? Then give me the API design.

**Aayush:** The comment and like records for a post can be queried, but this would increase feed load time, and the numbers can be rolled up into the Post entity for quicker rendering of the home feed.

API (identity taken from JWT auth token):

- `POST /posts` — Req: {text, mediaDetails}. Res: presigned S3 URL to upload media if media is part of post, else 2xx indicating post created. Idempotency key header to dedup client-side retries. The client uses the presigned URL to upload media directly to S3.
- `POST /follow/:userId` — Res: FollowRelationship(followingUserId, userId). Idempotency key header.
- `GET /posts?cursor={}&limit={}` — Res: Post(id, mediaUrl, text, likeCount, commentCount, createdAt)[], nextCursor:"". Cursor-based pagination to allow for dynamic insertion of posts in the feed.
- `POST /posts/:id/likes` — Res: 2xx indicating post liked.
- `POST /posts/:id/comments` — Req: {commentText}. Res: 2xx. Idempotency key header.
- `GET /myPosts?page={}&limit={}` — Res: Post(...)[]. Simple pagination works because checking self posts.

**Interviewer:** Three probes: (1) `POST /posts` returns a presigned URL, so the post row exists before the media does — what does a follower's feed show if the client crashes and never uploads? (2) Idempotency key on posts/follow/comments but not likes — deliberate? (3) Read your endpoint list against your own FRs — what's missing?

**Aayush:**
1. The post will have a status attribute, initially UPLOADING_MEDIA. For generating feeds only posts with status UPLOADED will be considered. So if the client fails to upload the media the status remains UPLOADING_MEDIA and it is excluded from others' feeds.
2. The like has uniqueness on (userId, postId) so client retry will fail at DB level.
3. Everything is covered.

**Interviewer:** 1 and 2 are good. Who flips UPLOADING → UPLOADED — the client doesn't call your API again. And on 3 I don't agree: your feed returns commentCount, but which endpoint serves the actual comments? Can a user ever undo a like or a follow?

**Aayush:**
1. S3 will fire a notification which will be the trigger to change the status of the Post record.
3. Yes there should be an endpoint to list the likes and comments for a post, and also delete endpoints for like, comment and follow relationship.

**Interviewer:** Give me those concretely — verb, path, response shape. Then move into the high-level design.

**Aayush:** `GET /posts/:id/likes?cursor={}&limit={}`, similarly for post comments. `DELETE /posts/likes/userId`, similarly for comments. `DELETE /follows/:userId`.

*(API phase complete — 36.0 min elapsed. Entities and API rendered to canvas.)*

**Interviewer:** Now the HLD — components, datastores, and directional flow for both paths.

**Aayush:** The client hits the POST endpoint for creating a post. The request passes through an API gateway which handles auth. The request is handled by the API service which creates a Post record (with media-uploading status) in a PostgreSQL DB — using Postgres because of its support for indices enabling quick querying of posts; the trade-off is additional complexity at write time, but our post write load is not very high so Postgres should suffice. The API service returns a presigned URL to the client. The client uses this presigned URL to upload the media. If the media is very large we can use multipart upload, saving the upload session in Postgres to support resumable uploads.

**Interviewer:** Now the read path. `GET /posts` for a user who follows 500 accounts and wants the 10 newest posts across all of them — component by component.

**Aayush:** The client hits the GET endpoint, handled by the API service, which queries the DB to fetch all the users the current user follows (FollowRelationship table indexed on followingUserId) and then fetches their posts (indexed on userId+createdAt). This will increase latency of feed generation and also put the main SQL DB in the hot read path, overloading it. We could have read replicas to reduce read load, but the actual querying will still blow up the latency budget for feed gen. To meet our latency budget we will have to precompute feeds for each user, and store these feeds in an external in-memory Redis cache. This brings feed-gen latency within budget and removes the DB from the hot path. The additional complexity of maintaining the cache comes in — the cache needs to be kept consistent with new posts, so we can have a CDC mechanism on the SQL DB triggering workers which update the Redis cache. Redis can have a TTL of 1 day to prevent memory usage from growing. If Redis goes down the read load falls on the DB; to avoid overwhelming it we can use request coalescing at the API service.

*(HLD phase complete — 49.4 min elapsed. HLD rendered to canvas.)*

**Interviewer:** Deep dive. Your feed workers do fan-out-on-write into Redis. A user with 50 million followers posts a photo. Walk me through what CDC → worker → Redis does in that moment, and what breaks.

**Aayush:** This is the common celebrity problem. Fan-out on write will not suffice because of the high follower count — the CDC workers will have to update feeds of 50M users/Redis keys on one post write. This can be solved by tracking which users cross a particular followerCount threshold and classifying them as celebrities. For celebrity users, in the write path there's no need to push to each follower. Instead on the read path, for each user we check the celebrities they follow using the FollowRelationship table and the standard list of celebrity users, then fetch the posts of the celebrity users and combine them with the pre-generated feed obtained from Redis. This combined feed is returned to the client. So we overcome the write fan-out by taking on complexity at read time.

**Interviewer:** (1) Your `GET /posts` uses a cursor. The feed is now a merge of two sources — what exactly *is* the cursor, and how do you guarantee page 2 doesn't repeat or drop posts? (2) A user crosses the celebrity threshold — 999k followers yesterday, 1.1M today. What happens to their posts in followers' feeds at that moment?

**Aayush:**
1. The cursor will consist of postId + createdAt from the celebrity feed and the Redis feed.
2. Any new posts they make will follow the complex path of being merged at read time, but previous posts are already present in the Redis pre-generated feed. So after they become a celebrity, the merge of the two lists should be a kind of union where duplicates are dropped.

**Interviewer:** Size it. (1) Fan-out write rate at peak, picking an average follower count. Is that a fleet you can run, and if not what gives? (2) Redis memory for 100M users' precomputed feeds — and the conclusion you draw.

**Aayush:**
1. Assuming average follower count of 1000, there are 10^8 writes/s to Redis at peak.
2. Assuming 1 post → 1KB, and each feed has 50 posts, total memory in Redis = 100M × 50 × 1KB = 5 TB. One single instance will not be able to hold all the data — we would need a Redis cluster sharded by userId.

**Interviewer:** Redis sizing is fine. On #1, redo the multiplication — peak posts/s × 1000 followers. I don't think 10⁸ falls out. Then: whatever the number is, plus 20k likes/s and 5k comments/s roll-ups on the same path — is that a fleet you'd run? If not, what lever do you pull?

**Aayush:** Did not understand, could you clarify the situation.

**Interviewer:** (a) Peak is 5 × 2k = 10k posts/s, each fanning out to 1000 followers. 10,000 × 1,000 = ? (b) Your design fans out every post to every follower's cached feed. Name one change to that policy that cuts the number substantially — think about who those 1000 followers actually are and whether all of them need a feed in Redis right now.

**Aayush:** a. 10^7. b. All users need a Redis entity for their feed and since the feed entry must be kept consistent we need to write to all the users' feeds.

**Interviewer:** You set a 1-day TTL on the Redis feeds. If a follower hasn't opened the app in a week, is their feed key even in Redis? So what is your worker accomplishing when it writes to that key?

**Aayush:** So an optimisation could be to write to feeds of only those users who have been active recently.

**Interviewer:** (1) Roughly what fraction does that eliminate? (2) A dormant user comes back after a week — their Redis key is gone. What does `GET /posts` do, and how does it interact with the coalescing/DB-fallback story?

**Aayush:**
1. Assuming 50% of followers are not very active for any user, fan-out gets cut by 50%.
2. `GET /posts` will query the DB for this particular user, generate the feed and update the Redis entry for the user. The initial feed load would take some time.

**Interviewer:** Operability. It's 2am and users say feeds are "missing posts." (1) What metric tells you the pipeline is falling behind, and what's your alert threshold? (2) CDC → worker lag at peak — what happens to UX, and what's the automatic mitigation and the manual lever?

**Aayush:**
1. The delay between the post creation and the post being written to Redis is the metric which tells how far the feed is lagging behind.
2. The user sees posts uploaded by the people they follow on a delayed schedule because of the lag in the workers. This agrees with the eventual consistency requirement for posts. The automatic mitigation would be to scale up the worker nodes to process the events faster, based on the CDC queue depth.

**Interviewer:** Closing question — the single biggest bottleneck left at 5× peak, and one decision you'd reconsider if the follower graph were 10× denser (10,000 average followers). Give me the alternative and what it costs.

**Aayush:** The likes and comments are the biggest bottleneck that have been left behind.

**Interviewer:** Take the second half too. 10× denser graph → 10⁸ Redis writes/s at peak. Which decision do you reconsider, and what does the alternative cost?

**Aayush:** With a denser graph the write load on Redis will be high for each post. We can't remove the write feed update path because otherwise the feed read latency will blow up.

*(Round ends — 78.0 min elapsed.)*

---

## Design Summary

**Requirements Gathered:**
- FRs: upload photo/video with caption; follow users; reverse-chronological home feed; like & comment; profile grid. Out of scope: analytics, recommendation/ranking.
- NFRs: 99.9% availability for upload; feed gen p99 < 200 ms; eventual consistency for posts/likes/comments with read-your-own-writes for posts; durable media storage.
- BoE: 100M DAU · 2 posts/user/day → 2k posts/s avg, 10k peak · 5 feed loads/user/day × 10 posts → 50k post reads/s · 1 MB/post → 2 GB/s ingest, ~60 PB/yr · 20k likes/s · 5k comments/s · Redis feed cache ~5 TB · peak fan-out 10⁷ writes/s.

**High-Level Architecture:**
Client → API Gateway (auth) → API Service. Write: create Post row in Postgres with `status=UPLOADING_MEDIA`, return presigned S3 URL; client uploads media directly to S3 (multipart + resumable, upload session in Postgres); S3 event notification flips status to UPLOADED. Read: precomputed per-user feeds in a Redis cluster sharded by userId (TTL 1 day), kept fresh by CDC on Postgres → feed workers → Redis. Read replicas plus request coalescing as the Redis-failure fallback. Hybrid fan-out: non-celebrity posts fanned out on write to active followers only; celebrity posts pulled at read time and merged (union with dedupe) against the cached feed, paginated by a composite (postId, createdAt) cursor. Cache miss → DB-backed feed rebuild that repopulates Redis.

**Key Design Decisions & Trade-offs:**
- Postgres for posts/graph — chosen for index support; cost acknowledged as write-time complexity, judged acceptable at 2k posts/s.
- Precompute feeds in Redis rather than query-on-read — removes the DB from the hot path, accepts cache-maintenance complexity.
- CDC → workers to keep the cache consistent.
- Cursor pagination for the feed (live insertion) vs offset pagination for the profile grid (own posts, stable).
- No idempotency key on likes — the unique (userId, postId) constraint makes retries naturally idempotent.
- Hybrid fan-out — write fan-out traded for read-time merge complexity.
- Fan out only to recently-active users — ~50% reduction, justified by the 1-day feed TTL.

**Scalability & Fault Tolerance Points:**
- Redis at 5 TB requires a sharded cluster, not a single instance.
- Redis down → read replicas + request coalescing at the API service.
- Feed pipeline lag monitored as post-creation → Redis-write delay; auto-scale workers on CDC queue depth.
- Object storage with hot/cold tiering for the 60 PB/yr media growth.

**Gaps / Missed Areas:**
- **No CDN and no media read path at all** — ~50 GB/s of media egress served directly from S3 in the design as drawn. Largest cost and latency item in the system, never raised.
- **No transcoding pipeline** despite video being in FR1 — no rendition ladder, no queue, no DLQ.
- **Like/comment count roll-up left as the "remaining bottleneck"** with no mechanism — no sharded counters, no batching; a viral post is row-lock contention on a single Postgres row.
- **Counter increments are not idempotent** under at-least-once CDC — never raised, despite correctly handling idempotency on the Like record itself.
- Celebrity fan-out required prompting; the 10× scale-break was left explicitly unsolved.
- Arithmetic slip: stated 10⁸ fan-out writes/s where 10⁷ falls out.
- No alert threshold numbers; no cost figure for the Redis cluster or S3/CDN; no handling of a single Redis shard failure or correlated cold-miss thundering herd.
- Deletes/unfollow effect on cached feeds never addressed; deleted-post visibility in cached feeds not covered.
- Delete/list endpoints produced only under prompting and without response shapes; `DELETE /posts/likes/userId` is not a well-formed path.

---

## Feedback Given

**Time Taken: 78 minutes.**

### 0. Pace report

| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 19.7 min | +11.7 | ✗ |
| Core entities | 12 min | 22.5 min | +10.5 | ✗ |
| API design | 17 min | 36.0 min | +19.0 | ✗ |
| High-level design | 27 min | 49.4 min | +22.4 | ✗ |
| Deep dive | 40 min | 73.7 min | +33.7 | ✗ |
| Wrap-up | 45 min | 78.0 min | +33.0 | ✗ |
| **Total** | **45 min** | **78 min** | **+33** | ✗ |

**Would this have fit a real 45-minute round? No — and not narrowly.** A real interviewer would have cut you off *mid-HLD*: at the 45-minute mark you had just finished describing the read path and had not yet touched fan-out. **The entire deep dive would never have happened** — no celebrity problem, no cursor merge, no Redis sizing, no operability. Those were the strongest 25 minutes of your round and in a real interview they don't exist.

Your single biggest time sink: **requirements ran 20 minutes**, and three of those were me asking the same retention question three times before you engaged with it. Second sink: the API phase took 19 minutes over reference because the "what's missing?" round-trip took four exchanges.

Blunt version: the design you produced is a hire. The design you'd have produced in 45 minutes is a no-hire, because it ends at "we precompute feeds in Redis" with zero hard problems touched.

### 1. Senior-signal scorecard

| Signal | Read | Why |
|---|---|---|
| **Owns the narrative** | **Mixed** | Genuinely good moment: on the read path you volunteered naive → "this blows the latency budget" → precompute → "here's the new complexity" → CDC → "and if Redis dies, coalescing." That's unprompted break/fix and it's the best thing you did all round. But every other trap needed me: celebrity fan-out, the merged cursor, the missing endpoints, the active-user optimization. |
| **Leads with trade-offs** | **Mixed** | Postgres was justified with an alternative-ish framing and an explicit cost. Cursor-vs-offset was justified *and* you correctly said offset is fine for the profile grid — a real trade-off, well made. But Redis, CDC, and read replicas were all asserted with no named alternative. |
| **Pushes scale until it breaks** | **Weak** | You never pushed scale yourself — I supplied the 50M-follower case, the fan-out multiplication, and the 10× graph. And when the break finally arrived (10⁸ writes/s) you stopped at "we can't remove the write path" with no alternative. That's the exact moment the round was asking you to design, and you declined it. |
| **API as a designed contract** | **Strong** | Best API you've produced. Idempotency keys on the mutating endpoints, a reasoned exception for likes via the unique constraint, presigned-URL upload, explicit field lists, cursor vs offset each justified. Docked slightly: the delete/list endpoints came only after prompting and arrived without response shapes, and `DELETE /posts/likes/userId` isn't a real path. |
| **Operability** | **Mixed** | Named the right lag metric and the right auto-mitigation (scale workers on CDC queue depth). But no alert *threshold* — I asked for a number and got prose. |
| **Pace** | **Weak** | 78 minutes for a 45-minute round, deep dive starting at minute 49. |

**Overall read: mid-level execution with senior flashes.** On content, a **hire**. On the clock a real loop runs, a **no-hire** — because the two things that would have earned the senior title (the feed break/fix and the celebrity merge) landed 30+ minutes past when they'd have been heard.

### 2. Performance Rating: 3/5

Competent, coherent design with a genuinely good self-driven break/fix on the read path and the strongest API contract to date. Held back by: needing prompts for every trap after the first, a 10× arithmetic slip on the fan-out rate, the final scale-break left explicitly unsolved, one entirely missing subsystem (CDN/media delivery), and a 1.7× clock overrun.

### 3. What a senior strong-hire would have done on THIS problem

**The gap that should worry you most: there is no CDN, and no media delivery path at all.** Your own numbers say 50k post reads/s × ~1MB = **~50 GB/s of media egress**, all served straight out of S3. That's the largest cost line in the system and it never came up. A senior designs the read path for the *bytes*: CDN in front of S3, signed CDN URLs, and the observation that feed JSON is small and cacheable while media is huge and static — different tiers for different objects. Also absent: **no transcoding**. FR1 says video; uploaded video needs a transcode pipeline (bitrate/resolution ladder), a long-running-task pattern with a queue, worker pool and DLQ — and it's the *real* reason a post needs an UPLOADING status, not just crash recovery.

**Traps that should have been self-raised:**
- **Celebrity fan-out** — for an Instagram question this is *the* known problem; raising it while describing fan-out is a free senior signal.
- **The like-count roll-up is a hot-key write problem** — 20k likes/s concentrated on viral posts means row-level contention on one Postgres row; batch/aggregate in Redis and flush, or shard the counter, accepting approximate counts for seconds.
- **Denormalized counts drift** — under at-least-once CDC, `count = count + 1` is not idempotent. The same trap you correctly dodged on the Like *record* via the unique constraint, you walked into on the *counter*.

**Alternative-justifications skipped:** why CDC rather than the app publishing to Kafka on write (CDC couples you to Postgres internals but survives a crash between commit and publish); why a Redis list rather than a ZSET scored on a time-sortable postId (which makes the merge, the dedupe and the cursor natural); why Postgres at all for an append-only, time-ordered, never-joined access pattern — Cassandra is the textbook alternative and you should name it and then say why you're not taking it.

**The exact scale-break and its fix:** at 10⁸ fan-out writes/s the answer isn't "we can't remove the write path." It's to **invert the ratio** — push the celebrity threshold down so far more accounts go read-path, converging on fan-out-on-read with a small precomputed core, paid for with read-path CPU and bought back with a short-TTL merged-feed cache so a scrolling session pays the merge once, not per page. The senior move is to say the *policy is a tunable dial*, not a binary.

**Operability missed:** no alert threshold number; no cost figure for a 5 TB Redis cluster or 60 PB of S3 (you identified the storage problem and never priced the fix); nothing on a single Redis *shard* failing rather than all of Redis; nothing on the thundering herd when a celebrity posts and 10M dormant feeds miss simultaneously — your coalescing answer was scoped to total Redis failure, not correlated cold misses.

**Credit where due:** the composite `(postId, createdAt)` cursor across two merged sources, and recognizing that the celebrity-threshold transition creates duplicates requiring union-with-dedupe — a sharp answer to a question most candidates fumble. It arrived at minute 63.

### 4. Checklist

Re-read `system_design_senior_guidance.md` — specifically §3 (push scale until it breaks) and §6 (pace). Content is close to the bar; the clock is not. Run the pre-round self-check's last line — *"Am I on track to finish core design by ~30 min?"* — against an actual timer next round.
