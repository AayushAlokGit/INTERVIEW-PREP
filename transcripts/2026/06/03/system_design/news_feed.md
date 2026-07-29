# System Design Round Transcript
**Date:** 2026-06-03
**Start Time:** 10:01
**End Time:** 10:53
**Duration:** 52 minutes
**Problem:** Design a News Feed System (Twitter/Facebook-like)

---

## Conversation Log

**Interviewer:** Design a news feed system — Twitter/Facebook-like. A user opens the app and sees a feed of recent posts from the people they follow, most-recent-first; users can also create posts. Start with requirements gathering.

**Aayush:** "I have solved this before."

**Interviewer:** Familiarity raises the bar rather than lowering it — expect you to drive crisply and reach the celebrity-fanout problem faster. Same expectations. Go ahead with requirements.

**Aayush:** Start time 10:01.

**Aayush:** "Do we need media support in posts?"

**Interviewer:** Yes — text plus optional media (images, short video). You decide how deep to go; media storage/CDN can be a bounded side concern with the feed as the core.

**Aayush:**
FRs: (1) create posts with optional media, (2) follow other users, (3) see a feed of recent posts from people/pages followed. Out of scope: likes, comments.
NFRs: (1) Highly available post creation (99.9%, ~8.6 hrs/yr downtime), eventual consistency OK for posts; (2) low-latency feed gen (p99 < 200ms); (3) low-latency media rendering; (4) 1B DAU, 1 post/day + 10 posts read/day → ~100k post-reads/s avg, ~10k writes/s avg; ~10KB/post → ~10TB/day storage growth; (5) handle celebrity posts fanning out to 10M follower feeds.

**Interviewer:** Pressure-tested two things: avg vs peak multiplier, and whether "read" means a post or a feed-load.

**Aayush:** Peak 3×. 2 feed reads/day, each feed = 10 posts → 20 posts read/day per user.

**Interviewer:** So provisioning ~300k post-reads/s and ~30k writes/s at peak, ~23k feed-requests/s avg (~70k peak). Move to core entities.

**Aayush:** Core entities: Post, User, FollowRelationship, PostMedia. Fields added as design progresses.

**Interviewer:** Move to API design — concrete endpoints, request/response shapes, pagination.

**Aayush:**
1. POST /posts — Req { postText, optionalMediaContentDetails }; Resp presigned S3 URLs (if media) OR 200 OK. (Idempotency key header added after prompt.)
2. GET /posts?nextCursor=&limit= — Resp { posts: [{ postId, authorId, authorName, text, mediaUrls, createdAt }], nextCursor: "ts_postId" }; cursor-based, composite (timestamp, postId).
3. POST /follow/:targetUserId — Resp { followRelationshipId }.

**Interviewer:** Probed feed response field shape and a retry failure case on POST /posts.

**Aayush:** Added explicit feed fields and an idempotency-key header to dedupe retries.

**Interviewer:** Move to high-level design — write path and read path, concrete components and datastores.

**Aayush (write path):** Client → API Gateway (auth) → Social Media Service. Service checks for media; if present, generates presigned S3 URLs and returns them. Creates Post and PostMedia records (PostMedia has s3Url + status=pending, FK postId). Client uploads media directly to S3 (bytes bypass servers). On upload completion, S3 event notifications flip PostMedia status to uploaded. SQL DB sharded on userId to handle ~30k writes/s peak. Same service handles POST /follow, creating FollowRelationship records.

**Interviewer:** Read path — how do you assemble a feed of recent posts from everyone the user follows, with posts sharded on userId?

**Aayush (read path):** Naive: FeedService queries SQL for the users followed, scatter-gathers their posts across shards, merges, returns. Optimize with indices (FollowRelationship on followerId; Post on authorId+createdAt; PostMedia on postId). But at 20k feed reads/s the queries are too slow and the DB bottlenecks; read replicas don't absorb the read volume. So precompute feeds: on each post creation, update precomputed feeds of all followers. Storing precomputed feeds in DB still bottlenecks reads, so move them to an in-memory distributed Redis cache. Maintain consistency via CDC events on post creation consumed by cache-update workers. Use a Redis cluster for availability.

**Interviewer:** Push scale until it breaks — compute the fleet-wide cache-write rate of fanout-on-write (state an avg follower count), then walk through a single celebrity post with 10M followers. Does the design survive?

**Aayush:** Traded read latency for write volume. Avg 1000 followers → 1 post = 1000 cache writes. Fine for normal users. For celebrities with millions of followers, workers would have to update millions of precomputed feeds — minutes of work. Solution: promote users to "celebrity" past a follower threshold; for celebrities, skip write fanout and do read-time joining instead, trading read latency to avoid write fanout.

**Interviewer:** Defend the hybrid. Walk the read-time merge concretely (where celebrity posts come from, how merged, p99 < 200ms with ~50 celebrities followed), and the hot-read concentration on a celebrity's posts.

**Aayush:** At read time query DB for celebrities followed (~50), fetch ~100 posts each (~5000 posts), merge-sort with the already-sorted Redis precomputed feed in the Feed Service, return to client. But that's still ~20k celebrity-post requests/s hitting the DB, so store celebrity posts in Redis too and read them from cache. The Redis node holding a celebrity's posts can become a hot node overwhelmed with reads; handle via replication across multiple nodes with the client randomly choosing a replica to spread read load — at the cost of maintaining replicated celebrity data.

**Interviewer:** Last deep-dive round — operability. (1) Worker lag symptom, SLA, and how you detect feed staleness. (2) What's stored in the precomputed feed and the total Redis memory cost across 1B users.

**Aayush (1):** User won't see the latest post due to worker lag — acceptable under eventual consistency, with a 1-min freshness limit. Alert when worker lag > 1 min. Measure lag per worker as (now − post.createdAt), take the max across workers as fleet lag.

**Interviewer:** Good — concrete SLA + metric + alert. Refinement: also track CDC offset lag to catch a stuck worker. Now part 2.

**Aayush (2):** Store postIds + mediaS3Urls + createdAt + text (~100KB), 50 posts/user → ~5MB/user → ~1PB, sharded across Redis nodes via consistent hashing.

**Interviewer:** Recheck the sizing — a feed entry is ~0.5–1 KB, not 100 KB (100 KB is the S3 image). 50 × ~1KB = ~50 KB/user × 1B ≈ ~50 TB, not 1 PB. Also flagged the denormalized-content choice vs IDs-only. Asked for end time.

**Aayush:** 10:53.

---

## Design Summary
**Requirements Gathered:** FRs — create posts (optional media), follow users, view most-recent feed; likes/comments out of scope. NFRs — 99.9% availability on writes, eventual consistency for posts, p99 < 200ms feed gen, 1B DAU, ~100k reads/s & ~10k writes/s avg (3× peak → ~300k/~30k), ~10TB/day storage, celebrity fanout to 10M.

**High-Level Architecture:**
- Write: Client → API Gateway (auth) → Social Media Service → sharded SQL (on userId); presigned-URL direct-to-S3 media upload; S3 event notifications flip PostMedia status pending→uploaded.
- Read: hybrid fanout. Normal users — fanout-on-write to precomputed Redis feeds, maintained by CDC-driven cache-update workers. Celebrities (promoted past a follower threshold) — no write fanout; their recent posts cached in Redis (replicated across nodes, reads spread across replicas) and read-time merge-sorted into the precomputed feed by the Feed Service.

**Key Design Decisions & Trade-offs:**
- Fanout-on-write (push) over fanout-on-read for normal users — trades higher write volume for low read latency.
- Hybrid pull model for celebrities — trades read latency for avoiding millions of writes per post.
- Precomputed feeds in Redis over DB — removes DB from the hot read path.
- Celebrity posts in Redis + replication + read-spreading — mitigates hot node.
- Consistent-hashing shard for the feed tier.

**Scalability & Fault Tolerance Points:** Sharded SQL on userId; read replicas considered and rejected as insufficient; precompute + cache for read scale; hybrid fanout for celebrity write amplification; hot-node mitigation via replication; CDC-driven cache consistency; 1-min freshness SLA with per-worker lag metric and alert.

**Gaps / Missed Areas:**
- BoE error: feed-entry size off by ~20× (100KB vs ~1KB) → 1PB vs ~50TB.
- Stored denormalized content without naming the IDs-only alternative/trade-off.
- Celebrity promotion/demotion cutover (existing pushed posts + backfill) and threshold flapping/hysteresis untouched.
- Read-time merge over-fetch (5000 entries for a 10-item page) not bounded to top-K.
- Idempotency key only added after prompt.
- CDC offset lag (vs only time-lag) and a DLQ for failed fanout writes not raised.

---

## Feedback Given

### Per-criterion
- **Requirements clarification — Strong.** FRs/NFRs with numbers unprompted, out-of-scope explicit, celebrity case flagged early. One nudge each for peak multiplier and read-unit disambiguation.
- **Core entities — Adequate.** All four named up front; deferred fields; ordering key on Post (the design-driving field) could have been named earlier.
- **API design — Strong (one reflex to fix).** Composite cursor pagination, explicit response shape, presigned two-phase upload. Idempotency only after a retry prompt.
- **High-level architecture — Strong.** Clean write path, concrete datastores, named shard key, direct-to-S3 + event-driven status.
- **Deep dive — Strong (highlight).** Drove naive→break→fix repeatedly under own power; self-raised celebrity fanout → hybrid, and hot-node → replication.
- **Scalability & fault tolerance — Strong.** Push/pull trade-off, hybrid, hot-key mitigation, consistent hashing, freshness SLA.
- **Operability — Mixed (up).** Strong lag SLA + metric + alert; missed offset-lag and promotion/demotion flapping.
- **Communication — Strong.** Structured, self-driven, minimal hint-seeking.
- **Diagram Quality — Strong.** Faithfully tracked the verbal design (requirements, entities+fields, API contract, write and read paths) with directional flow; no word-vs-box gaps.
- **Main miss — BoE arithmetic.** ~20× sizing error (1PB vs 50TB); must land within an order of magnitude at this level. Denormalized-vs-IDs trade-off stated silently.

### Senior-signal scorecard
| Signal | Rating | Reason |
|---|---|---|
| Owns the narrative / self-raises traps | Strong | Drove naive→break→fix; self-raised celebrity fanout and hot-node unprompted. |
| Leads with trade-offs vs named alternatives | Mixed | Push-vs-pull framed well; IDs-only-vs-denormalized and SQL/Redis choices not justified vs alternatives. |
| Pushes scale until it breaks | Strong | Took fanout-on-write to the 10M break and designed the hybrid. |
| API as a designed contract | Strong | Composite cursor + explicit shape; idempotency after nudge. |
| Operability / second-order concerns | Mixed (up from Weak) | Strong lag SLA/metric/alert; missed offset-lag and threshold flapping. |
| Pace | Mixed | 52 min; overage spent in the deep dive, not early. |

**Overall: Senior — Hire (borderline strong-hire).** Best round in the set; narrative ownership and self-driven push to the hybrid are senior behaviors. Held back from clean strong-hire by the order-of-magnitude BoE error and trade-offs stated as choices.

### What a senior strong-hire would have done on THIS problem
- Stated the ~10M cache-writes/s fleet number explicitly to justify the cache tier.
- Sized the feed tier to ~50 TB and used it to *choose* IDs-only over denormalized (consistency on edits/deletes).
- Self-raised celebrity promotion/demotion cutover + backfill + hysteresis around the threshold.
- Bounded the read-time merge to top-K recent per celebrity (avoid 5000-entry over-fetch for a 10-item page).
- Added CDC offset lag + a DLQ for failed fanout writes.

### Self-drill
Review `system_design_senior_guidance.md`, especially item #2 (every box gets a "...and here's what I'm trading away") and the BoE line in the pre-round self-check.
