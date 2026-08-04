# System Design Round Transcript
**Date:** 2026-08-04
**Start Time:** 15:20:39 · **End Time:** 16:29:25 · **Duration:** 68.8 min (budget 45)
**Problem:** Product Recommendation System (item-page related products + personalized homepage feed)
**Performance Rating:** 2/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->

> **Clock note:** only the start (15:20:39) and the stamps from 16:07:09 onward were real `Get-Date` reads. The interviewer estimated the intermediate stamps (15:23–15:42) instead of reading the clock, and they drifted. Phase actuals below are marked accordingly. Aayush caught the drift himself mid-round; all five interview skills were subsequently amended to forbid guessed timestamps.

## Phase Timings
| Phase | Budget | Actual | Hit? |
|---|---|---|---|
| Requirements | 8 min | ~est. 7 min (not reliably stamped) | Roughly |
| Core entities | 12 min | ~est. 10 min (not reliably stamped) | Roughly |
| API design | 17 min | ~est. 16 min (not reliably stamped) | Roughly |
| High-level design | 27 min | started ~18 min, still incomplete at 68 min | **No** |
| Deep dive | 40 min | 57–68 min, partial (scale-break surfaced, never quantified) | **No** |
| Wrap-up | 45 min | 68.8 min | **No** |

Single largest cost: one turn between roughly 15:42 and 16:07 — **~25 minutes of silence on one question**.

---

## Conversation Log

**Interviewer:** [15:20:39] Presented the problem — recommendation infrastructure for a large e-commerce marketplace. Two surfaces: (1) item-page "customers who viewed this also viewed", (2) personalized homepage feed for signed-in users, refreshed as they browse. Signals: views, clicks, add-to-cart, purchases, searches. Ranking model is a black box; the round is about infrastructure. Gave the canvas path and the 45-min phase budget. Asked for requirements.

**Aayush:** FRs: 1. Show related product for a product in the product page. 2. Homepage — show ranked list of recommended products for a signed user (refreshed as they browse). 3. Collect user activity events like views, clicks, add-to-cart, purchases and searches to generate recommendations. NOTE: actual ranking of products done by an ML model (abstracted); the model is fed the user data and product data.

**Interviewer:** Asked for NFRs with numbers — traffic, latency, freshness, availability, consistency — plus what is explicitly out of scope.

**Aayush:** NFRs: 1. low latency feed generation (p99 < 200ms). 2. Up to 3 min update time for recent events to affect homepage. 3. Assuming 100M DAU and 10^5 s in 1 day, each user viewing 10 products a day → 10k events/s ingested (peak 10x). Each opens homepage 3 times a day → 3000 homepage reads/s avg (peak 10x).

**Interviewer:** Rendered FRs/NFRs to the canvas. Noted the arithmetic checks out. Three gaps: (1) FR1's read surface was never sized — how many product-page requests, and how does that compare to 3k/s; (2) availability and consistency targets, and whether a stale recommendation costs what a wrong payment costs; (3) what is out of scope.

**Aayush:** 1. 10k read requests/s emitted for similar products to a product. 2. Highly available (99.9 ~ 8.2 hours of downtime/year) and eventual consistency acceptable for recommendations.

*(Out of scope skipped. 99.9% is 8.76 h/year, not 8.2.)*

**Interviewer:** Flagged the skipped item and moved to core entities — what objects exist and which fields matter.

**Aayush:** Core Entities: 1. UserActivityEvent (userId, type, time, metadata..) 2. UserHomepageFeed (userId, products:[]..., updatedAt) 3. SimilarProducts (productId, similarProducts:[].., updatedAt)

**Interviewer:** Three probes: (1) no ellipses — what is actually in `metadata` and `products[]`, IDs or objects, and what does that cost at read time; (2) there is no `Product` entity, but a strip needs title/image/price/stock — is `Product` in this system, and if not what does that imply for the read path; (3) `UserActivityEvent` is keyed by user but FR1 is co-view driven — what identifies the item, and how are anonymous viewers handled. Then asked for the API.

**Aayush:** 1. products is a list of IDs; at read time the IDs can be used to extract the actual product bodies — trades off read latency for storage. 2. Product (id, name, details..) 3. UserActivityEvent will also have a productId optional column populated when the event is in context of a particular product.

*(Anonymous viewer not addressed.)*

**Interviewer:** Noted the logged-out viewer was still unanswered; asked him to fold it into the API. Demanded verbs, paths, request/response shapes, pagination.

**Aayush:**
```
NOTE: Identity extracted from JWT token if needed

GET /products/:id/relatedProducts   — public, no auth
Response: Product(id, name, title, price)[]

GET /user/:id/recommendedProducts
Response: Product(id, name, title, price)[]

POST /userActivityEvents
Request body: {type, productId, time}
Idempotency key header for de-duplication in event of client retries.
Response: 2xx status indicating event accepted.
```

**Interviewer:** Credited the unprompted idempotency key. Three probes: (1) neither read endpoint pages — mechanism and why; (2) `:id` in the path *and* identity from JWT — what happens when they disagree; (3) the event body cannot identify an anonymous viewer, so FR1 drops all logged-out traffic. Then asked for the HLD.

**Aayush:** 1. cursor based pagination to allow for dynamic insertions. 2. no need for /user/:id — simply recommendedProducts would be good. 3. we can create a general user to represent all users who visit pages without logging in.

**Interviewer:** Rendered entities + API to the canvas. Parked a challenge: if every logged-out visitor is the *same* general user, all anonymous views collapse into one stream and you can no longer tell that person A viewed X then Y — what does that do to FR1's co-view signal? Asked for the HLD: write path and read path separately, every component, every arrow labelled.

**Aayush:** we can use the IP address of the request to track the uniqueness of the request in case of no logged in user. This would be part of the POST request body.

**Interviewer:** Noted it and pushed for the HLD.

**Aayush:** clients would fire the POST API to send activity events. In order to ingest the events we need a DB which would support high write throughput — Cassandra, because it appends to a WAL and provides fastest writes. However reads get complex; to optimise the event read we could partition the user activity data by userId, and add a timebucket as part of the partition key to avoid hot shards. We can then use eventTime as sort key.

**Interviewer:** Said that is the event store but there is still no path from an event to a rendered recommendation. Asked: what consumes the events, where and on what trigger the model is invoked, where `UserHomepageFeed` and `SimilarProducts` live, what serves the two GETs, and the click → homepage path inside 3 minutes.

**Aayush:** do we need to consider the events only for a particular window when generating recommendations for a user?

**Interviewer:** Answered: yes — model input is the user's recent activity, ~30 days of history with the current session weighted most heavily; older activity archived. Asked him to finish the HLD.

**Aayush:** your time has drifted from actual time why is this ?

**Interviewer:** [16:07:09 — real read] Acknowledged the error: only the first stamp had been a real `Get-Date` read; the rest were estimates. True elapsed 46.5 min, budget already gone, and roughly 25 minutes of it was a single turn. Asked for the remainder of the HLD in one message.

**Aayush:** fix the skills to never guess the time

*(Round paused. All five interview skill files amended with an explicit rule requiring a live `Get-Date` in the same turn as any timestamp written; committed as 608aa81. Round then resumed.)*

**Aayush:** where were we ?

**Interviewer:** [16:09:23] Summarised state: FRs/NFRs/entities/API locked, HLD half-built (ingest into Cassandra, nothing downstream), and listed the four missing pieces plus the two parked threads.

**Aayush:** need hint for the consumer of event

**Interviewer:** [16:10:41] Declined the hint. Reframed as four questions instead: who needs an event after it is stored and in what shape; which component computes co-view pairs and is it per-event or batch; what does a 3-minute target rule in and out; and — the load-bearing one — what query would a consumer issue against a store partitioned by `userId` + time bucket to get "everything in the last minute".

**Aayush:** a cron job would schedule worker instances every 3 mins which would generate the new user homepage feed by querying the user activity events of the required timeframe from Cassandra (querying would be easy because of userId + time based partitioning) and passing the product data to the ML model which would generate the list of recommended products for the user. The recommendation can be stored in a simple SQL store to allow quick querying using indices on userId. The SQL can be queried to get the homepage feed for a user. At peak load a simple DB will not be able to handle the read load so we introduce a Redis cache in front of the DB to remove DB from the hot path. We can use cache-aside strategy with request coalescing to prevent stampedes. This Redis would be kept consistent by CDC mechanisms for when the SQL store homepage record is updated. Similar flow for the similar products.

**Interviewer:** [16:18:39] Credited cache-aside + coalescing + CDC. Then pushed to break it, asking for arithmetic: (1) how many users does the cron regenerate, what does 100M / 3 min imply in model invocations/s; (2) write the Cassandra query for "all events in the last 3 minutes" — which partitions, how many, lookup or scan; (3) the same number is the SQL write rate and the Redis invalidation rate — does SQL survive; (4) "similar flow for similar products" is wrong, because co-view is cross-user and cross-session while the store is partitioned by userId.

**Aayush:** need to group the events by session and not user so sessionId + timebucket. 1. Now cron will regenerate feed for only the users who have had some activity in the last 3 mins. 2. The cron job would query Cassandra for the events across all sessions for last 3 mins. 3. *(blank)*

**Interviewer:** [16:23:55] Credited both corrections. Pressed (2): Cassandra can only fetch a partition whose key you know; the worker does not know which `sessionId`s were active — how does it enumerate them, how many partitions is a scan, what does a time secondary index cost at 100k writes/s, and if neither works, what does that say about whether the event store is the right thing for a worker to read from at all? Re-asked for the active-user number and the SQL write rate.

**Aayush:** (2) with sessionId, timebucket will need to check all partitions for the last 3 mins data — scatter gather across all shards. Better to have all the events sent to a Kafka queue, partitioned by sessionId.

**Interviewer:** [16:28:13] Confirmed that was the right fix and the best self-driven move of the round. Asked for the last three, needing numbers: active users per 3-min window → model invocations/s at avg and 10× peak; the same figure as SQL write rate and Redis invalidation rate; and what Cassandra is still for now that Kafka exists, plus what consumes the topics.

**Aayush:** lets move to feedback

**Interviewer:** [16:29:25] Round closed.

---

## Design Summary

**Requirements Gathered:**
- FRs: related products on a product page; personalized homepage feed for signed-in users; ingest of view/click/add-to-cart/purchase/search events. Ranking model abstracted.
- NFRs: p99 < 200 ms; 3-min freshness; 10k events/s ingest (peak 10×); 3k homepage reads/s; 10k product-page reads/s; 99.9% availability; eventual consistency acceptable.
- Out of scope: never stated despite being asked twice.

**High-Level Architecture (as described):**
Clients → `POST /userActivityEvents` → **Kafka** (partitioned by `sessionId`; a late correction replacing a direct write/read against the event store) → **Cassandra** event store (partition `sessionId` + timeBucket, sort key `eventTime`) → **cron every 3 min** → **worker instances** (only for users active in the window) → **ML model** → **SQL store** for `UserHomepageFeed` (index on `userId`) → **CDC** → **Redis** (cache-aside, request coalescing) → GET endpoints. "Similar flow" asserted for `SimilarProducts` but never designed.

**Key Design Decisions & Trade-offs:**
- Cassandra for ingest, justified by WAL-append write throughput; no named alternative offered.
- Time bucket added to the partition key specifically to avoid hot shards — self-raised.
- Feeds store product IDs only, hydrated at read time — explicitly framed as trading read latency for storage.
- SQL for feeds with an index on `userId`; Redis in front to keep the DB off the hot path; cache-aside with request coalescing against stampedes; CDC for cache consistency.
- Idempotency key on the event write — self-raised.
- Cursor pagination "to allow for dynamic insertions".
- Self-corrected partitioning from `userId` to `sessionId` once co-view was pressed.
- Self-corrected from cron-over-Cassandra to Kafka once scatter-gather was pressed.

**Scalability & Fault Tolerance Points:**
- Hot shards addressed at ingest via time bucketing.
- Read scaling addressed via Redis + coalescing.
- Scatter-gather identified and fixed with Kafka.
- Nothing on DLQ, backpressure, consumer lag, replication, failover, or fallback when cache and DB are both unavailable.

**Gaps / Missed Areas:**
- **No quantification of the scale break.** Asked four separate times for active-users-per-window → model invocations/s; never produced. This was the senior-differentiating conversation and it never happened.
- Out-of-scope never stated (asked twice).
- The last item in a list left blank three separate times.
- `SimilarProducts` computation (co-view pair counting, normalisation, session capping) never designed — waved off as "similar flow".
- Anonymous identity: general-user → IP; NAT/CGNAT and mid-session IP change never addressed.
- Product hydration path never designed even though the ID-only decision made it load-bearing.
- No named alternatives for Cassandra, SQL, or Redis.
- No monitoring/alerting, no cost estimate, no cold start, no consumer lag or backpressure.
- 99.9% stated as 8.2 h/year (actual 8.76).

---

## Feedback Given

### Round conditions
- **68.8 minutes against a 45-minute budget**, with the HLD still incomplete at the end. About 25 of those minutes were a single turn.
- **One hint requested** (consumer of the event stream) — declined; you then produced the answer yourself, which is the point.
- Clock error was mine: intermediate stamps were estimated, not read. You caught it. The skills are fixed.

### Rubric

**Requirements clarification — 3/5.** The best phase. You derived 10k events/s and 3k reads/s independently with correct arithmetic and stated peak multipliers unprompted — that is a genuine improvement. Deductions: you sized only the homepage surface and had to be asked for the product-page read volume, which is the larger one; 99.9% is 8.76 h/year, not 8.2; and out-of-scope was asked for twice and never given.

**Core entities — 2/5.** Thin, and delivered with ellipses (`metadata..`, `details..`) that hide exactly the fields that decide the design. `Product` only appeared when I pointed out that a recommendation strip needs a title and a price. No `Session` entity even after you moved your whole partitioning scheme onto `sessionId` — the entity list never caught up with the architecture.

**API design — 3/5.** The idempotency key on the write path was raised unprompted, and dropping `:id` from the recommendations path in favour of JWT identity was the right instinct on being challenged. But pagination only appeared when asked; `Product(id, name, title, price)` has both `name` and `title` and no image or stock flag; there is no error contract, no versioning, no notion of a stable cursor across regeneration, and putting the client's IP in the request body is a spoofable identity.

**High-level architecture — 2/5.** Never finished. The ingest half was well-reasoned; the serving half arrived only under direct questioning; the consumer/compute half never arrived at all. `SimilarProducts` — one of your two functional requirements — was dismissed as "similar flow" and never designed, when it is a genuinely different computation.

**Component design & trade-offs — 2/5.** You state choices, not trade-offs. "Cassandra because it appends to a WAL and provides fastest writes" names no alternative and concedes nothing. Same for SQL, same for Redis. The one exception was excellent: IDs-versus-bodies, where you explicitly said you were trading read latency for storage. That sentence is the template — every major box needs one.

**Scalability & fault tolerance — 2/5.** You pre-empted hot shards with a time bucket, unprompted, which is a real senior tell. But the actual break in your design — 100M feeds regenerated every 3 minutes — I had to surface, and even after four requests you never did the division. No DLQ, no backpressure, no consumer lag, no failover, no fallback path.

**Deep dive quality — 2/5.** There was one, and it was the strongest stretch of the round: pushed on how a worker enumerates partitions, you reasoned to "scatter-gather across all shards" and then to "Kafka partitioned by sessionId" without being told. That is exactly right and you got there yourself. But it arrived at minute 65 with nothing left after it, and the naive → break → fix → trade-off loop only ran once, driven entirely by my questions.

**Communication — 2/5.** Three times you answered items 1 and 2 of a three-item list and left the third blank. Out-of-scope was asked twice and never answered. The anonymous-user question took three attempts. And 25 minutes of silence on one turn is, in a real interview, the round ending.

**Diagram quality — 3/5.** The canvas matches what you said, including the revisions, because I drew only what you named. It is legible and directional up to the ingest and serving edges. What it shows most clearly is the hole in the middle: there is no consumer between Kafka and the feed store, and no `SimilarProducts` computation anywhere.

---

### Senior-signal scorecard

| Signal | Status | Reason |
|---|---|---|
| Owns the narrative (self-raises traps) | **Mixed** | Self-raised idempotency, hot shards via time bucketing, and cache stampedes — all real. But asked for a hint on the consumer, and every scale trap had to be handed to you. |
| Leads with trade-offs vs alternatives | **Weak** | Cassandra, SQL and Redis all chosen without naming what they beat or what they cost. One good exception (IDs vs bodies). |
| Pushes scale until it breaks | **Weak** | Asked four times for active-users-per-window → invocations/s and never produced a number. The break was surfaced by me, not you, and never quantified. |
| API as a designed contract | **Mixed** | Idempotency key and JWT-over-path-id are senior moves; pagination only on request, no shapes, no errors, no versioning, spoofable IP identity. |
| Operability / second-order concerns | **Mixed** | Cache stampede, coalescing, CDC, hot shards — genuinely good instincts. Zero monitoring, cost, lag, backpressure, DLQ, cold start, or fallback. |
| Pace | **Weak** | 68.8 min against 45, HLD unfinished, 25 minutes on one turn. |

**Overall read: mid-level. Against a senior bar this round is a no-hire** — not because the instincts are wrong (several are genuinely good) but because the design had to be extracted, and the one conversation that separates senior from mid — *what breaks at this scale and what number proves it* — never happened despite four invitations.

### **Performance Rating: 2/5 — Weak.** Eligible for re-ask.

---

### What a senior strong-hire would have done on THIS problem

**Done the division without being asked, in the first two minutes of the HLD.** 100M DAU over 10⁵ s means roughly `100M × 180/10⁵ ≈ 180k` users active in any 3-minute window. So dirty-user-only regeneration is ~1k model invocations/s, ~10k at peak — and *full* regeneration would have been `100M/180 ≈ 550k/s`, which is the number that kills the naive design. You reached the right answer ("only users with activity") by instinct but never computed either figure, so you couldn't demonstrate that the fix was a 500× reduction. The number is the argument.

**Self-raised the cron-over-a-database problem.** The moment you write "a scheduled worker queries the store for the last 3 minutes," a senior asks *what partition key does that query have?* — and answers "none, it's a scatter-gather, so the DB is the wrong source; I need a log." You got there, but only after being pushed twice. That is the difference between designing and being led.

**Designed `SimilarProducts` as its own problem.** It is not "a similar flow." It is: sessionize the stream by `sessionId`; emit co-viewed pairs (A,B) per session; cap session length or a k-view session emits k² pairs and a crawler blows up your job; aggregate in a window; and then *normalise* — raw co-view counts just rank the catalog by popularity, so you need lift or PMI, `count(A,B)/(count(A)·count(B))`, or every product recommends the iPhone. None of this resembles the per-user feed path.

**Split candidate generation from ranking, out loud.** You cannot score 50M products per request or per user-window. Cheap recall — ANN over embeddings, plus the co-view lists of recently viewed items, plus category trending — narrows to ~1–2k, and only then does the expensive model rank. Without that split, "pass the product data to the ML model" has no bounded cost.

**Killed the SQL store on its merits.** Access is a single-key lookup of an opaque blob. No joins, no transactions, no range scans. A KV store is strictly better, and saying *why* — naming SQL as the alternative and what it buys you (nothing here) — is the trade-off sentence you didn't produce.

**Noticed write-through beats cache-aside here.** Your feed generator is the *only* writer. It can populate the cache directly and you can delete the entire CDC pipeline. Cache-aside plus CDC is the right answer when many writers exist; recognising that yours has exactly one is the second-order thought.

**Rejected IP for anonymous identity in one line.** CGNAT puts thousands of unrelated people behind one address — which merges strangers' co-views — and a mobile IP changes mid-session, which splits one person's. Session cookie plus device ID, and accept that cookie loss means lost identity.

**CDN'd the product-page endpoint.** `/products/{id}/similar` is not personalized. It is the same payload for every visitor and it is your largest read surface at 10k/s. A CDN with a 5-minute TTL removes most of that load before it reaches your service — and it falls out of a fact you already established (it's a public, unauthenticated endpoint) without any new machinery.

**Owned "how do I know it's broken."** Feed staleness histogram (`now − generatedAt`, p99 alerted at the 3-minute SLA), Kafka consumer lag as the leading indicator, coverage (% of DAU with a fresh feed), fallback rate, and model version pinned per feed row so a bad rollout is attributable. You designed a system with a freshness SLA and no way to observe whether it is being met.

The full optimal design — entities, API with shapes, two-plane compute architecture, the trade-off table and the deep dives — is on the canvas below your own section.

**Checklist:** re-read `system_design_senior_guidance.md` before the next round. Item 3 ("push the scale a notch and confront what breaks") and item 6 (pace) are the two that decided this round.
