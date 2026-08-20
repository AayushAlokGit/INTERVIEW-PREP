# System Design Round Transcript
**Date:** 2026-08-20
**Start Time:** 16:03:58 · **End Time:** 16:25:45 · **Duration:** 22 min
**Problem:** Post Search
**Difficulty:** 4/5 (Hard) — an unbounded, forever-retained corpus where the naive index partitioning breaks under scatter-gather, and the fix (time bucketing + early termination) immediately creates a second break (write hotspot) plus a third (rare-term walkback); three dependent hard things in one dive
**Dominant pattern:** scaling reads (search fan-out) with a scaling-writes hotspot on the hot partition
**Performance Rating:** 4/5  <!-- machine-read on future rounds; <=2 = eligible for re-ask, >=3 retired -->

**Would it have fit a real 45-min round?** Yes — finished at minute 22, deep dive from minute 15, nothing truncated

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 10 min | +2 | Close |
| Core entities | 12 min | 13 min | +1 | Yes |
| API design | 17 min | 13.5 min | -3.5 | Yes |
| High-level design | 27 min | 15 min | -12 | Yes |
| Deep dive | 40 min | 20.5 min | -19.5 | Yes |
| Wrap-up | 45 min | 22 min | -23 | Yes |
| **Total** | 45 min | 22 min | -23 | Yes |

## Round-trip Tax
| Phase | Parts asked | Answered 1st pass | Follow-up exchanges | Minutes lost | What was missing |
|---|---|---|---|---|---|
| Requirements | 2 | 1 | 2 | ~3 | NFRs needed a separate prod; scale givens requested |
| Core entities | 1 | 1 | 1 | ~1 | Ordering field (createdAt) absent until probed |
| API design | 1 | 1 | 0 | 0 | — |
| High-level design | 1 | 1 | 0 | 0 | — |
| Deep dive | 4 | 3 | 1 | ~0 | Freshness / monitoring never answered |
| **Total** | 9 | 7 | 4 | ~4 of 22 | — |

**Deferrals used:** 0 — nothing needed deferring in a round with this little probing.

---

## Conversation Log

**Interviewer:** Design Post Search. Users on a large social platform write short text posts. Design the system that lets any user search across all posts and get back relevant results. **Difficulty: 4/5 (Hard)**. [45-min reference timeline given, measured not enforced; canvas path given] Start with requirements — FRs first, then NFRs with numbers.

**Aayush:** How will the users search for the posts — by keywords in the post, or by the user who has created the post?

**Interviewer:** Free-text keyword search over the post body. A query is one or more words; a post matches only if it contains all of the query terms. Searching by author is not required.

**Aayush:** FRs -> 1. Users can create short text posts. 2. Users can issue multiple free text keywords in a query and posts matching all the query terms are returned. Out of scope -> 1. Searching posts by author.

**Interviewer:** NFRs — with numbers.

**Aayush:** What scale does the system need to handle?

**Interviewer:** 500M DAU · 2 posts/user/day · 300 B average post · retained forever · 5B searches/day · peak 3x average. Convention: a day is 10^5 seconds. QPS, storage, index size and anything else derived is yours.

**Aayush:** NFRs -> 1. Highly available for post searching (99.99). Lower availability for post creation acceptable (99.9). 2. Posts should be durably stored. 3. Eventual consistency acceptable for post search. 4. Low latency post search (p99 < 500ms). 5. 500M DAU with 2 posts/user/day (300 B per post) -> 10k posts/s avg, peak 3x -> 3MB/s -> 300GB/day ~ 90 TB/year. 5B searches/day -> 50k searches/s, assuming each search returns 2 posts -> 100k post reads/s. 10:1 read:write ratio.

**Interviewer:** [rendered to canvas] Core entities.

**Aayush:** Core Entities -> 1. User (id, name, email) 2. Post (id, text, userId) — inverted index on text

**Interviewer:** A query for ["world","cup"] matches 50 million posts. You return one page. What decides which posts the user sees first, and which field on Post carries it?

**Aayush:** createdAt is missing — that would be the ordering field.

**Interviewer:** API design.

**Aayush:** POST /posts · body -> {text:string} · Response -> 201 {postId}. GET /search?terms={terms}&pageSize=&cursor= · Response -> 200 {posts:[{postId, text, userId, createdAt}], nextCursor}. Errors -> 400 for invalid request, 401 for unauthenticated request, 429 for rate limited requests, 500 for internal server error. Identity from JWT.

**Interviewer:** [rendered to canvas] High-level design.

**Aayush:** Client -> API Gateway (routing, auth, rate limiting). API Gateway -> Post Service for post creation. Post Service -> writes to Post DB (Cassandra, partitioned by postId). Post Service -> publishes postCreated event to Kafka. Kafka -> Indexer Service (consumer group). Indexer Service -> tokenizes text, writes to Search Index (Elasticsearch). API Gateway -> Search Service for search. Search Service -> queries Search Index. Search Index returns postIds. Search Service -> hydrates postIds from Post DB (with a Redis cache in front). Search Service -> returns results.

**Interviewer:** [rendered to canvas] Deep dive. Your index holds every post ever written — 1B a day, retained forever. Take the query ["world","cup"] at peak. Walk me through what actually happens inside that Elasticsearch cluster: how is the index partitioned, how many partitions does that one query touch, and what does each of them do?

**Aayush:** The index is partitioned by postId ranges — each shard holds an inverted index for the posts it owns. A query for two terms is a scatter-gather: the coordinator fans out to all shards, each shard looks up the posting list for "world" and for "cup", intersects them, sorts its local matches by createdAt desc, and returns its top pageSize. The coordinator merges the per-shard top-k and returns the global top pageSize.
For sizing: 300 GB/day of raw text, and the inverted index is roughly the same order as the raw text, so call it 300 GB/day of index. A shard should stay around 50 GB to keep query latency sane, so that is 6 new shards every day, ~2000 shards/year, and after a few years we are in the tens of thousands of shards. Every query touches all of them, so at 150k peak QPS the fan-out is 150k x 20000 shard-hits per second. That does not work.

**Interviewer:** That break is the round. Fix it.

**Aayush:** Shard by time instead of postId. Each shard holds a contiguous createdAt range — say one shard per day, or per few hours. Since results are ordered by createdAt desc, the coordinator does not need to fan out to everything. It queries the newest shard first, and only walks back to older shards if it has not filled pageSize yet. For a common term like "world cup" the newest shard alone fills the page, so a query touches one or two shards instead of twenty thousand.
Cost: rare-term queries degrade. A query whose terms only appear in a handful of old posts has to walk back through many shards before it fills a page, worst case all of them. And writes are no longer spread — every new post lands in the newest shard, so that one shard takes the entire 10k/s write load while every older shard is read-only.

**Interviewer:** Take your own write hotspot. At peak, every post on the platform lands on one shard while the other twenty thousand sit read-only. Fix it — and tell me what the fix costs you on the read path.

**Aayush:** Split the newest time bucket into N sub-shards and hash posts across them by postId. So the time dimension picks the bucket, the hash spreads writes inside it. With N=32 the newest bucket takes 30k/s spread over 32 nodes, ~1k/s each, which is fine.
Cost on the read path: a query now has to hit all N sub-shards of a bucket instead of one, so the fan-out per bucket goes from 1 to 32. That is still 32 instead of 20000, so it is a good trade, but it is not free — p99 is now the slowest of 32 responses instead of one, so tail latency gets worse.
Older buckets can be compacted back down — once a bucket stops taking writes, merge its 32 sub-shards into fewer larger ones, so the fan-out cost only applies to the hot bucket.

**Interviewer:** You raised the rare-term case and left it. Someone searches ["zbigniew","kayak"] — a pair that appears in maybe forty posts, all of them years old. What does your system do, and what does the user experience?

**Aayush:** The coordinator walks back bucket by bucket from newest to oldest and finds nothing until it hits year-old data. Worst case it scans all 20000 buckets, each scan is a posting-list lookup that returns empty, and the user waits far past 500ms — blowing the p99 NFR on exactly the queries that look cheapest.
Fix: keep a per-bucket term filter — a bloom filter or just the set of distinct terms per bucket — in memory on the coordinator. Before dispatching to a bucket, check whether the bucket could contain all the query terms; if not, skip it without a network call. Skipping is then a memory lookup, not an RPC, so walking 20000 buckets costs microseconds instead of seconds.
Cost: the filter has to be kept in sync with the index, and bloom filters give false positives so you still occasionally dispatch a useless query. Memory: 20000 buckets x a few MB of filter each is tens of GB, so the coordinator tier needs to be a dedicated fleet with that resident, or the filter itself gets sharded.

**Interviewer:** A post is created. When can it be found by search — and how do you know, in production, when that stops being true?

**Aayush:** done

**Interviewer:** [round ended; question recorded as unanswered]

---

## Design Summary

**Requirements Gathered:**
- FR1 create short text posts · FR2 free-text multi-keyword query, AND-matching. Out of scope: search by author.
- NFR: 99.99 search availability / 99.9 create · durable posts · eventual consistency for search · p99 < 500ms.
- Traffic: 10k posts/s avg, 300 GB/day, "~90 TB/year"; 50k searches/s; "2 posts per search" -> 100k post reads/s; 10:1 read:write.

**High-Level Architecture:**
Client -> API Gateway (routing, auth, rate limiting). Write path: Gateway -> Post Service -> Cassandra (partitioned by postId) and -> Kafka `postCreated` -> Indexer Service consumer group (tokenizes) -> Elasticsearch. Read path: Gateway -> Search Service -> Elasticsearch (terms -> postIds) -> hydrate from Redis cache, falling back to Cassandra -> results.

**Key Design Decisions & Trade-offs:**
- Recency ordering on `createdAt` rather than relevance ranking (decided silently, never stated as a scoping choice).
- Index partitioned by **time bucket** rather than postId range, so recency ordering permits early termination: newest bucket first, walk back only if the page is unfilled. Turns 20,000 shard-hits per query into 1–2.
- Hot bucket **hash-subsharded into N=32** to spread the 30k/s peak write load. Cost named: fan-out per bucket 1 -> 32, and p99 becomes the slowest of 32 responses.
- Cold buckets **compacted** back into fewer shards once they stop taking writes, confining the fan-out penalty to the hot bucket.
- Per-bucket **term/bloom filters resident on the coordinator** so a rare-term walkback skips buckets with a memory lookup instead of an RPC. Costs named: filter/index sync, false positives, and tens of GB resident -> dedicated coordinator fleet or a sharded filter.

**Scalability & Fault Tolerance Points:**
- Full self-driven break->fix chain: scatter-gather across 20k shards at 150k peak QPS -> time bucketing + early termination -> write hotspot -> hash subsharding + cold compaction -> rare-term walkback -> per-bucket filters. Three breaks, all fixed, every cost named.
- Fault tolerance essentially absent: no DLQ on the indexer, no behaviour stated for an ES write rejection, no failover, no consumer-lag position, no answer for an indexer dying mid-batch.

**Gaps / Missed Areas:**
- Ranking/relevance never raised as a design axis despite the prompt saying "relevant results".
- No freshness NFR (how soon a post must be searchable) — the requirement the whole async index path exists to satisfy.
- No monitoring of any kind; the closing freshness/observability question went unanswered.
- No idempotency key on `POST /posts`; no dedup on the at-least-once Kafka -> ES path.
- No position on cursor stability while 10k posts/s arrive mid-pagination.
- `2 posts per search` asserted and unchecked — understates hydration load by 5–10x.
- "peak 3x -> 3 MB/s" mislabels the average as the peak (peak is 9 MB/s); "~90 TB/year" vs the correct ~110 TB/year.
- 150k peak search QPS never stated in NFRs, though used correctly later in the deep dive.
- No named alternative for Cassandra, Kafka, Elasticsearch or Redis.
- No cost discussion despite "retained forever" at 110 TB/year, and no storage tiering for cold buckets.

---

## Feedback Given

**Requirements.** FRs adequate but thin — two requirements plus one out-of-scope item, and the right opening question (keyword vs author) before anything was written. Never asked what "relevant" means: the prompt said "relevant results" and he converted that to strict AND-matching plus recency ordering without raising ranking as a design axis. Defensible as a product decision (it is Twitter's "Latest" tab), but made silently — said out loud it is scoping, left unsaid it reads as not having noticed.

NFRs: right categories (availability split by path, durability, consistency, p99), two number problems. `10k posts/s` and `300 GB/day` correct; `"peak 3x -> 3 MB/s"` mislabels the average as the peak (peak is 9 MB/s); `"~90 TB/year"` should be ~110 TB; `50k searches/s` correct but the 150k peak was never stated in NFRs, though used correctly later; and `"assuming each search returns 2 posts"` is asserted, implausible (a page is 10–20 results) and understates the hydration path by 5–10x — that number then fed the read:write ratio. The NFR never written: **freshness** — how soon a post must be searchable — which is the requirement the entire async index path exists to satisfy, and why the last question of the round had nowhere to land.

**Core entities — 4/5.** Two entities, correct granularity; `createdAt` produced immediately when the ordering field was probed rather than defended. Calling out the inverted index at entity level was good instinct.

**API design — 4/5.** Strongest phase relative to history: request shapes on the write, cursor pagination with `nextCursor` and `pageSize`, a full 400/401/429/500 error table volunteered for the first time, JWT identity stated unprompted. Gaps: no idempotency on `POST /posts` (a retry on a timed-out create produces two posts, and the downstream Kafka path is at-least-once with no dedup key at either end), and no position on cursor stability while 10k posts/s arrive mid-pagination.

**High-level design — 4/5.** Clean and conventional in the right way; every component earned its place and the write path is correctly decoupled from the index path. Missing: resilience of any kind — no DLQ on the indexer, no behaviour on an ES write rejection, no failover, no consumer-lag position. Cassandra "partitioned by postId" named with no alternative compared.

**Deep dive — 5/5.** His best deep dive to date. He found the break himself with no prompting: sized the index, chose a shard size, derived shard count, projected forward, multiplied by peak QPS, and concluded it does not work — the full naive->break arc, volunteered. The fix (time bucketing with early termination) is exactly the insight the problem is built around, and both of its costs were named in the same breath. The hotspot fix went beyond the standard answer in two ways: the honest read-path cost (*"p99 is now the slowest of 32 responses instead of one"* — most candidates say "32x fan-out" and stop at the arithmetic), and **compacting cold buckets** so the fan-out penalty is confined to the hot bucket. The rare-term answer was complete and, crucially, **he sized the dependency he introduced** — tens of GB resident, therefore a dedicated coordinator fleet or a sharded filter — which is a standing weakness he cleared unprompted. The one thing not done: the freshness/monitoring question, which went unanswered entirely.

**Communication — 4/5.** Dense, structured, self-driving; conclusions stated before justifications. Cost: three questions got one-word non-answers (`done`, `next question`), including the final one, and "next question" arrived while his own scale break was still open — he had to be sent back to it.

**Pace.** 22 minutes total against a 45-minute reference, deep dive from minute 15, nothing truncated — a complete reversal of the standing pattern (last sitting: 56 min, deep dive starting at minute 45). Honest caveat recorded: the three deep-dive answers arrived roughly a minute apart, each several hundred words with the trade-off pre-named, a different working rhythm from the rest of the round where NFRs took ten minutes and the ordering field needed a probe. Scored as submitted; the pace number is provisional until reproduced on the clock, unassisted, on an unseen problem.

**Senior-signal scorecard.** Owns the narrative — **Strong** (derived the break unprompted with full arithmetic, volunteered both costs of his own fix; did not raise ranking, freshness or idempotency). Leads with trade-offs — **Mixed** (every deep-dive answer carried its cost, but Cassandra, Kafka, Elasticsearch and Redis all arrived bare). Pushes scale until it breaks — **Strong** (two full break->fix cycles, both self-initiated). API as a designed contract — **Strong** (all first-pass; missing idempotency and cursor stability). Operability — **Weak** (no DLQ, no failover, no cost, no monitoring, freshness question unanswered; a fully async index path with no observability story). Pace — **Strong**.

**Overall read: senior on architecture, mid-level on operations. Hire.** The scale break, the fix, the cost of the fix, and the fix for the cost is a four-step chain most candidates do not complete, and he drove all of it. What holds this below strong-hire is that the system as described cannot be operated: nothing measures index freshness, nothing catches a poisoned message, nothing reveals that search has quietly gone twenty minutes stale.

**Performance Rating: 4/5.** Retired. No hints used, no ceiling binds. Two clear gaps: the operability phase never happened, and the traffic model carries two unchecked assumptions a senior would have caught in the moment.

**What a senior strong-hire would have done here:** stated the ranking decision out loud in requirements instead of making it silently; written a freshness NFR ("searchable within 5 seconds") to make the index path measurable; sanity-checked `2 posts per search` in ten seconds and sized hydration at 500k–1M/s; named one alternative per datastore (Cassandra vs sharded relational, Elasticsearch vs hand-rolled inverted index on object storage, Kafka vs direct dual-write) — the asymmetry between excellent trade-offs on his own inventions and none on his dependencies is the whole of the Mixed grade; closed the async loop with indexing lag as a first-class metric (`now - createdAt` of the last indexed post, per partition), consumer-lag alerting, a DLQ, and dedup on `postId` at the ES write since Kafka offsets give at-least-once; put an idempotency key on `POST /posts` given his own 99.9% write availability implies ~9 hours of failure a year and therefore client retries; and costed the thing — tens of thousands of shards, tens of GB of resident filters, 110 TB/year forever, with cold-bucket storage tiering falling naturally out of the time bucketing already designed.

**Drill.** Write the operational runbook page for this exact design — four things only: the metric that tells you search is stale, the alert threshold on it, what the page says to check first, and what on-call does when the indexer is 40 minutes behind at peak. He architects at the bar; he does not yet describe systems as things that will be run at 3am by someone who did not build them.
