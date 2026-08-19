# System Design Round Transcript
**Date:** 2026-08-19
**Start Time:** 11:41:31 · **End Time:** 12:40:18 · **Duration:** 59 min (round content complete at 12:38:16 = 56 min)
**Problem:** Price Tracking Service
**Difficulty:** 3/5 (Medium) — one real scale break (serving 2 years of chart data under a 1s p99 at 200 rps), one genuine storage trade-off (sample vs change-point), and a polling fleet that must be sized against an external rate limit
**Dominant pattern:** scaling writes (poll ingest) + realtime updates (threshold notifications), with a scaling-reads break on the chart path
**Performance Rating:** 2/5  <!-- machine-read on future rounds; ≤2 = eligible for re-ask, ≥3 retired -->

**Would it have fit a real 45-min round?** No — cut off at minute 45, which is the exact minute the deep dive began. Everything that produced signal (550TB, sharding, hot reads, egress, rate limiting, push-vs-poll) would never have been reached.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 13 min | +5 | Over |
| Core entities | 12 min | 19 min | +7 | Over |
| API design | 17 min | 26 min | +9 | Over |
| High-level design | 27 min | 37 min | +10 | Over |
| Deep dive | 40 min | 56 min | +16 | Over |
| Wrap-up | 45 min | 56 min | +11 | Over |
| **Total** | 45 min | **56 min** | **+11** | Over |

Stamps: start 11:41:31 · scope Qs 11:44:44 / 11:46:07 / 11:46:32 · FRs 11:47:05 (+6m) · crawl Q 11:51:08 · scale Q 11:51:29 (+10m) · NFRs + scale deferral 11:55:00 (+13m) · aggregation Q 11:58:27 · entities 11:58:39 (+17m) · dedup answer 12:00:50 (+19m) · API v1 12:06:14 (+24m) · API fixes 12:07:22 / 12:07:50 (+26m) · givens rendered 12:09:26 (+27m) · HLD 12:18:27 (+37m) · notification Q 12:19:16 · sizing 12:26:30 (+45m) · re-derivation 12:28:54 (+47m) · storage 12:30:10 (+48m) · sharding 12:31:31 (+50m) · hot reads + p99 12:33:56 (+52m) · egress 12:35:16 (+54m) · rate limit + Black Friday 12:38:16 (+56m) · feedback 12:40:18

## Round-trip Tax
| Phase | Parts asked | Answered 1st pass | Follow-up exchanges | Minutes lost | What was missing |
|---|---|---|---|---|---|
| Requirements | FRs, NFRs+numbers, out-of-scope, read:write | 2 of 4 | 1 | ~0 in phase, ~16 later | Scale/traffic model (deferred), read:write ratio, out-of-scope left blank |
| Core entities | 4 entities | 4 of 4 | 1 | ~2 | Product identity / URL dedup — answered well when asked |
| API design | 3 endpoints | 1 of 3 clean | 3 | ~4 | Subscription id in path, payload-size position, `nextCursor` |
| High-level design | components + flows | most | 1 | ~1 | Who writes the price point to the SQL DB |
| Deep dive | 8 probes | 3 of 8 | 6 | ~14 | Chart-load QPS (2 asks), total storage, read hot keys, egress consequence |
| **Total** | | | **12** | **~21 min (38% of round)** | |

**Deferrals used:** 1 — the scale/traffic model, deferred at minute 13. He did **not** return to it voluntarily; it was handed back at minute 37 and then consumed minutes 37–53.

---

## Conversation Log

**Interviewer:** Design a Price Tracking Service. Users track products on third-party e-commerce sites, set a target price, get notified when price drops to/below target, and can view a price-history chart. **Difficulty: 3/5 (Medium)**. [reference timeline given, clock measured not enforced, canvas path given] Start with requirements.

**Aayush:** will the users specify which domains to watch a particular product for or must we scour the web to find all references to the desired product?

**Interviewer:** The user gives you a specific product URL on a specific retailer — one product, one site. No cross-web matching or identity resolution.

**Aayush:** what should be the time window for the price chart?

**Interviewer:** Up to 2 years of history, and the client can request any window inside that.

**Aayush:** how fine should the time intervals be?

**Interviewer:** That's a design decision, not a given — pick a granularity and justify it against your freshness requirement.

**Aayush:** FRs ->
1. Users can add products which they want to be tracked and URLs for product in e-commerce sites from where the tracking needs to be done.
2. Users can choose to be notified when the price of a product <= set target price.
3. Users can view product history as chart (any time window in last 2 years)

**Interviewer:** Good. Now the non-functional requirements — with numbers.

**Aayush:** do we need to crawl over the web to discover new links for a product?

**Interviewer:** No. No discovery, no crawling for links. The user hands you the exact URL; you only ever fetch URLs someone has already registered.

**Aayush:** what is the scale?

**Interviewer:** Givens — take these as fixed:
1. 100M registered users, 10M DAU
2. 20 tracked products per user on average
3. 50M distinct product URLs across all users
4. A product's price changes ~once per week on average
5. Chart views: 2 per DAU per day
6. Retention: 2 years of price history
7. Peak-to-average 3×; on Black Friday price-change rate goes 20× normal

Convention: a day is 10^5 seconds. Derived numbers — QPS, storage, poll rate, fanout — are yours.

**Aayush:** FRs -> [as above]

Out of scope ->

NFRs ->
1. Highly available for detecting price change (99.99). For price chart and user notification, lower availability acceptable.
2. Freshness latency for price change notification is 5 min. Price chart latency p99 < 1s.
3. Eventual consistency acceptable for price points data.
4. Price data should be retained for 2 years.

I will derive the scale at later point in the design

**Interviewer:** Noted — I'll hold you to the scale derivation. Requirements are on the canvas. Two things: your out-of-scope list is empty, and you haven't stated a read:write ratio. Move to core entities.

**Aayush:** is there any aggregation needed for the price points in the chart?

**Interviewer:** That's yours to decide. The product requirement is only that a user can render a chart for any window inside 2 years and it comes back under 1s — how you store and serve that is your design.

**Aayush:** Core Entities ->
1. User (id, name)
2. Product(id, productPageUrl, createdAt)
3. PriceSubscription(productid, userId, createdAt, targetPrice)
4. PricePoint(productId, price, timestamp)

**Interviewer:** One structural question before the API: two different users register the same Amazon URL. Do they get one `Product` row or two — and what does that decision do to your polling workload?

**Aayush:** for this design i will consider each product to have a unique url for it, so when 2 different users register for a same url they get linked to the same product. This will reduce the polling load since we need to only poll for one product and notify different users on it

**Interviewer:** Good — that's the right call and you named the reason. Move to API design.

**Aayush:** API Design

NOTE: Identity extracted from JWT auth token

POST /priceSubscriptions
Request: {productUrl}
Response: PriceSubscription(id, userId, targetPrice = null, productid)

PUT /priceSubscriptions/notify
Request: {targetPrice}
Response: PriceSubscription(id, userId, targetPrice = null, productid)

GET /products/:id/pricePoints?from={}&to={}
Response: PricePoint(productid, price, timestamp)[]

**Interviewer:** Two structural questions. (1) `PUT /priceSubscriptions/notify` — the body carries only `targetPrice`. Which subscription is being updated? (2) `GET /products/:id/pricePoints?from&to` returns a bare array. A user asks for the full 2-year window. What comes back over the wire, and what's your position on the size of that payload?

**Aayush:** 1. the URL must also have :id request param
2. Need cursor pagination

**Interviewer:** What's the response body then — spell it out.

**Aayush:** PricePoints()[], "nextCursor": timestamp

**Interviewer:** API is on the canvas. Move to the high-level design.

**Aayush:** render the scale givens on the canvas so that i can refer them

**Interviewer:** [renders givens block] Go ahead with the high-level design.

**Aayush:** we can have a cron job which will execute every 5 mins which is our freshness latency; it will get the price by querying the URL of the different products. The price point that will be obtained will be queried against the price subscriptions for the particular product id and then if needed the relevant users would be notified; finally the price point can be saved to a DB like SQL with indexing on the timestamp. This DB can be queried to serve the price chart.

This is a very simple design and it will break in many places like one cron job instance will not be able to scour 50M distinct products — for that we need to have a fleet of cron jobs and assign products to different cron instances. This would help scale the price point gathering for the extended products.

Now for one product price point, the cron job doing the target price subscription and price point saving could cause issues if the cron job goes down — we could either lose the price point or lose the notification. So the cron job must simply put the price point along with timestamp to a kafka queue to ensure durability of the price point. The kafka can be partitioned by productId to ensure price points for a product are ordered. Then consumers can process the price point event and decide whether to send notification or not.

**Aayush:** Regarding the notification what kinds of notification will need to be sent?

**Interviewer:** Push notification and email — assume a Notification Service already exists that you call with `(userId, channel, payload)`. Its internals are out of scope. HLD is on the canvas. One ambiguity I left marked in red: you moved the price-point write off the cron job and onto Kafka, but you didn't say who writes it to the SQL DB. Who does? And now make good on the deferral: **size the polling fleet.** 50M distinct URLs, 5-minute freshness — what's the fetch rate, and does that number hold up?

**Aayush:** it comes out to 200k requests/s that the fleet must make. These 200k product requests/s must be spread out across a fleet.
there can be 2 consumers for the kafka partition — one consumer will notify the users and other will update the SQL DB. Both the consumers need to be idempotent.

And regarding the SQL DB storage for 2 years there will be approx 220k price points (1 price point per 5 min), assuming each price point is 50B - 11 MB for price chart of 1 product.
Now for each chart load 220k price points will be fetched, and there are 20 chart loads/s so 440k price point reads/s and write is 200k price points writes/s.
Read:write ratio is almost 2:1

**Interviewer:** Two things. (1) Re-derive chart loads per second from the givens — show the arithmetic. (2) You're storing one price point every 5 minutes. Given #4 says a product's price changes about once a week. What exactly are you writing to that DB, and what does that choice cost you?

**Aayush:** 1. 10M DAU x 2 -> 20M chart loads/day -> 20 chart loads/s
2. I am writing redundant price points, since price will be constant for long periods this is costing me extra storage but the math says that storing all the price points at 5 min granular intervals does not cost much storage plus having finer granularity will allow to detect price changes quickly.

**Interviewer:** (1) 20M per day divided by 10^5 seconds — do that division again. (2) You said the storage doesn't cost much. You sized 11MB for *one* product. What's the total across all 50M?

**Aayush:** 1. 200 chart reads/s so read:write becomes 20:1
2. 550TB for 50M products

**Aayush:** so we would need to shard the SQL DB by productId, this will not have any effect on read queries since reads will be served from same partition. There is no hot partition problem here because the incoming price point traffic is always at fixed rate decided by the frequency of executing the cron job.

**Interviewer:** Two pushes. (1) You justified "no hot partition" using write traffic only. Reads land on the same shards — a product goes viral and a million users load its chart. Does that change your answer? (2) A single chart request for the full 2-year window reads how many rows from one shard, and you promised p99 < 1s. Does that hold?

**Aayush:** 1. yes read loads can be skewed and a popular product can have high read traffic; in this case we can have read replicas to distribute the traffic across multiple nodes.
2. A single chart request asks for 220k price points, which should be queryable within 1 s with a SQL DB; however if the latency appears to be crossing the threshold then we can introduce an in-memory cache for faster query times but this will introduce additional complexity of maintaining the cache consistent.

**Interviewer:** 11MB per chart response at 200 requests/second — compute the egress bandwidth that implies, and tell me whether the client actually needs all 220k points to draw a chart.

**Aayush:** ~2 GB/s egress. The client does not need the entire history, they just need an interval which they will specify and also there will be cursor based pagination.

**Interviewer:** Last two. (1) Your fleet is making ~170k HTTP fetches per second against a handful of retailer domains. What does Amazon do about that, and what does it mean for your design? (2) Black Friday: price-change rate goes 20×. Walk me through what happens to your pipeline and to your 5-minute freshness NFR.

**Aayush:** 1. Amazon can rate limit our IP address; in this case we must respect amazon rate limits, so we would have to add our own rate limiter before making external calls to get price points. We would have to maintain our own distributed rate limiting infra and corresponding configs as well.
2. The current design uses the polling approach for determining price changes; with this approach in order to catch more frequent price updates we will have to reduce the polling interval. But a better approach would be to pursue a push based approach where we could register a webhook with amazon and amazon would push price updates to our webhook.

**Interviewer:** That's the round. [feedback follows]

---

## Design Summary

**Requirements Gathered:**
- FR1 add product + URL to track; FR2 notify when price ≤ target; FR3 price-history chart over any window in 2 years.
- Out of scope: left blank.
- NFR1 99.99% availability on price-change *detection*, explicitly lower on chart + notification. NFR2 5-min freshness for notification, p99 < 1s for chart. NFR3 eventual consistency OK for price points. NFR4 2-year retention.
- Traffic model deferred at minute 13, produced at minutes 45–53 under prompting.

**Traffic model as finally derived:**
- 50M URLs / 300s = ~167k fetches/s (he said 200k — right order).
- 2 years at 5-min granularity = ~220k points/product; 50B each = ~11MB/product; **550TB total**.
- Chart loads: 10M DAU × 2 = 20M/day = **200/s** (initially derived as 20/s — 10× slip, corrected on second ask).
- Read:write stated as 20:1 (measured in requests). Measured in rows touched it is ~250:1 — the inversion was never caught.
- Egress at full-window charts: 11MB × 200/s = **~2.2 GB/s**.

**High-Level Architecture (as described):**
- Fleet of cron jobs on a 5-min interval, products statically assigned across instances, fetching prices directly from retailer product URLs.
- Cron publishes `{productId, price, timestamp}` to **Kafka, partitioned by productId** (per-product ordering); cron does nothing else, so a crash loses neither the price point nor the notification.
- Two consumers: one queries `PriceSubscription` by `productId` and notifies matching users; one writes the price point to a **SQL DB indexed on timestamp**. Both stated as needing idempotency.
- SQL DB sharded by `productId`; read replicas added for skewed chart reads.
- Chart served from the SQL DB via `GET /products/:id/pricePoints` with cursor pagination.
- Own distributed rate limiter in front of external fetches to respect retailer limits.

**Key Design Decisions & Trade-offs:**
- One `Product` per unique URL → polling cost is per-URL, not per-subscription. Reason stated unprompted. **Best decision of the round.**
- Cron → Kafka → consumers, rather than cron doing fetch+notify+persist, justified by durability under crash. Naive → break → fix, volunteered.
- Kafka partitioned by `productId` justified by ordering.
- Storing a sample every 5 min rather than on-change: acknowledged as redundant, kept for detection granularity, justified on a storage estimate that was never totalled at the time.
- In-memory cache floated for chart latency, with cache-consistency cost named — the one fully-formed trade-off of the round.
- Poll → push (retailer webhook) floated as the better long-term shape under Black Friday load.

**Scalability & Fault Tolerance Points:**
- Fleet-scale-out for the 167k/s fetch rate; static product assignment across instances.
- Shard SQL by `productId`; read replicas for hot-product chart reads.
- Kafka as the durability boundary between fetch and both downstream actions.
- Consumer idempotency raised unprompted.
- External rate limiting, with the operational cost (own distributed limiter + configs) named.

**Gaps / Missed Areas:**
- **Repeat notifications** — price sits below target, consumer fires every 5 minutes forever. Never raised by anyone. No `armed` flag / `lastNotifiedAt` / dedupe key on the user-visible action.
- **Sample vs change-point storage** — noticed the redundancy, dismissed it; the on-change alternative is ~500GB instead of 550TB (~1000×) and dissolves the chart-read break entirely.
- **Downsampling / rollups** — probed twice via payload size and egress; answered with pagination both times. Never reached "a chart is ~1000 pixels, so ~730 daily rollup points serve a 2-year window".
- **Datastore alternative never named** — "a DB like SQL" for 550TB of append-only time-ordered data, with no time-series store (Cassandra / ClickHouse / Timescale) named as the alternative, despite NFR3 already granting the consistency budget.
- **Notify-path hot key** — a viral product's million subscriptions are all scanned on every price point; no `(productId, targetPrice)` index / sorted-set range query.
- **Scheduler failure** — static assignment means a dead cron instance silently stales its products, breaking the 99.99% detection NFR. No lease/visibility-timeout work queue, no adaptive polling.
- **Operability** — no monitoring at all: no staleness metric, no consumer lag alerting, no DLQ for unparseable pages, no per-domain failure rate, no cost.
- **API** — no error semantics anywhere; no list/delete subscription endpoints; PUT response still says `targetPrice = null` on the endpoint that sets targetPrice.
- Out-of-scope section left empty; read:write ratio never stated in the requirements phase.
- Two 10× arithmetic slips (chart QPS, then the read:write ratio built on it).

---

## Feedback Given

### 0. Pace report

| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 13 min | +5 | Over |
| Core entities | 12 min | 19 min | +7 | Over |
| API design | 17 min | 26 min | +9 | Over |
| High-level design | 27 min | 37 min | +10 | Over |
| Deep dive | 40 min | 56 min | +16 | Over |
| Wrap-up | 45 min | 56 min | +11 | Over |
| **Total** | **45 min** | **56 min** | **+11** | **Over** |

**Would this have fit a real 45-minute round? No.** A real interviewer would have cut him off at **minute 45 — the exact minute his deep dive started.** Everything that produced signal — 550TB, sharding, hot reads, egress, retailer rate limiting, push-vs-poll — happened in the last eleven minutes and would never have been reached. What the interviewer would have walked away with: three endpoints, a cron-plus-Kafka diagram, and no traffic model.

**Biggest time sink: deferring the scale derivation.** Deferred at minute 13, not returned to voluntarily, handed back at minute 37, then consumed minutes 37–53 — a third of the round — in the phase where the deep dive was supposed to live. Deferring sizing doesn't save time, it moves it to the most expensive part of the clock.

This was a 3/5 Medium. **The overrun was process, not problem size.**

Also noted: four of five requirements-phase questions were scope clarifications; two of them ("how fine should the intervals be", "is aggregation needed") were his own design decisions, not givens — and both turned out to be the two decisions this problem hinges on.

### Round-trip tax
[table as above — 12 follow-up exchanges, ~21 min ≈ 38% of the round]

**Deferrals used: 1**, not returned to. That's the difference between a deferral and a skip.

### Rubric

**Requirements — Weak.** FRs clean and correctly scoped. NFRs had real numbers on latency and availability, and **tiering availability by path (99.99% on detection, lower on chart/notify) is a genuinely senior move** — most candidates give one blanket number. But no traffic model, no read:write ratio, out-of-scope left as an empty header; and when the numbers arrived 32 minutes later, two were wrong by 10×.

**Core entities — Strong.** Four entities, correct shape, `PriceSubscription` correctly modelled as the join carrying `targetPrice`. The URL-dedup answer was the best moment of the first half — identified that one `Product` per URL collapses the polling workload, and named the reason unled. Missing: nothing on `Product` tracks last-known price or next-poll time, which is why the poller can't tell whether anything changed.

**API design — Mixed.** Positives: JWT identity stated up front, request shapes on both writes, cursor pagination with `nextCursor`. Negatives, all needing a probe: no subscription id in the PUT path (resource not addressable); payload-size position only on request; no error semantics anywhere. Uncaught: the PUT response is copy-pasted from POST and still says `targetPrice = null` on the endpoint whose entire job is setting it. Also missing: no list/delete subscription endpoints.

**High-level architecture — Strong, and the best part of the round.** He did the thing the rubric most wants, unprompted: stated a deliberately naive design, named its breakages himself, fixed each. Single cron → fleet. Cron doing everything → cron publishes to Kafka so a crash loses neither price point nor notification. Kafka partitioned by productId with the reason stated. Two consumer groups, and **he raised that both must be idempotent before being asked.** Exactly the narrative-ownership signal — it just arrived at minute 37.

**Scalability & fault tolerance — Weak.** 167k fetches/s right order. 550TB correct, sharding conclusion follows properly. Read replicas correct. But the core traffic model was wrong twice in the same direction — 20M/day became 20/s instead of 200/s, and read:write came out 2:1 when measured in rows touched it's ~250:1. That inversion made him conclude this is write-heavy with a manageable read path; it is emphatically the opposite, and every downstream decision rested on the wrong premise.

**Deep dive — Weak.** The scale break was handed to him twice and never resolved. He computed 11MB/response and 2.2 GB/s egress correctly, then answered with cursor pagination — which makes it worse, since paginating 220k points means dozens of sequential round trips under a 1s p99. The question being asked — *does the client need 220k points to draw a chart* — has an answer he never reached, and it's the highest-leverage decision in the system.

**Communication — Good.** Structured, phase-disciplined, no dead air, self-corrected cleanly on both 10× slips without defending them.

**Diagram quality — Good.** Everything described was renderable without ambiguity, with one exception flagged in red: after moving persistence off the cron he never said who writes to the SQL DB.

### 1. Senior-signal scorecard

| Signal | Status | Why |
|---|---|---|
| Owns the narrative (self-raises traps) | **Mixed** | Volunteered naive→break→fix on ingest and raised consumer idempotency unprompted — genuinely strong. But deferred the scale model and never returned; needed probes for hot reads, payload size, rate limiting, retailer behaviour. |
| Leads with trade-offs vs alternatives | **Weak** | One full trade-off (cache speed vs consistency) and one acknowledged (redundant points vs detection latency). Otherwise bare: "a DB like SQL" with no time-series alternative, Kafka with none, sharding with no key comparison. |
| Pushes scale until it breaks | **Weak** | Every number came out under direct instruction; two 10× errors in the core model; handed the actual break (2.2 GB/s egress) he answered with pagination rather than confronting it. |
| API as a designed contract | **Mixed** | Request shapes, JWT identity, cursor + nextCursor present; but two of three endpoints needed a structural probe to become correct, no error semantics, and the PUT response contradicts the PUT's purpose. |
| Operability / second-order concerns | **Mixed** | Rate limiting answered well with the operational cost named; read replicas correct. But nothing on monitoring, DLQ, consumer lag, poll staleness, or cost — and no answer for a cron instance dying holding an assignment. |
| Pace | **Weak** | Every phase over; deep dive started at minute 45; 56 minutes total. |

**Overall: mid-level. No-hire at a senior bar**, hire at a mid bar. Architecture is coherent and ingest-path reasoning is genuinely good; what's missing is senior-shaped — driving his own numbers, justifying against alternatives, resolving the break instead of naming a mitigation beside it.

### 2. **Performance Rating: 2/5**

Deciding factors: the traffic model was wrong by 10× twice and produced an inverted read:write ratio the whole design rested on; the central scale break was surfaced twice and left unaddressed; the deep dive started at the minute a real round would have ended. The ingest-path work was strong enough to pull toward a 3, but the single hardest thing in this problem was never solved.

### 3. What a senior strong-hire would have done on THIS problem

**The trap he should have self-raised, and nobody mentioned all round: repeat notifications.** Poll every 5 minutes; a price drops below target and *stays* there; the consumer fires a notification every 5 minutes, forever, until the price rises. That's the most user-visible bug this system can have, and the design ships it. Fix: state on the subscription — an `armed` flag or `lastNotifiedAt`, cleared only when price rises back above target — plus a dedupe key `(subscriptionId, priceChangeId)` so at-least-once delivery doesn't double-fire. He raised idempotency *of the consumer*; this is idempotency *of the user-visible action*, the harder and more important one.

**The signal he walked past: store change-points, not samples.** He noticed the redundancy — "I am writing redundant price points" — then dismissed it because storage looked cheap. At one point per change instead of one per 5 minutes, two years is ~200 points/product, ~10KB, **~500GB total instead of 550TB**. Three orders of magnitude, and free — a price chart is a step function, so change-points *are* the complete information. Polling frequency and storage frequency are independent decisions and he collapsed them into one. Poll every 5 min for detection; write only when the value differs from last-known (Redis, keyed by productId); optionally a daily heartbeat so gaps are distinguishable from outages.

That one decision also dissolves the break he couldn't solve: a 2-year chart becomes ~200 rows, not 220k. p99 < 1s stops being a question. Egress drops from 2.2 GB/s to ~2 MB/s. For genuinely volatile products, add pre-computed daily rollups (min/max/open/close) so any long window returns ~730 points — a chart is ~1000 pixels wide, so more points than that is data you pay to ship and the client pays to discard.

**The alternative never named for the datastore.** "A SQL DB with an index on timestamp" holding 550TB of append-only time-ordered data is the wrong shape, and a senior names why in the same breath: write-once, never-updated, always range-scanned by `(productId, time)`. Cassandra with partition key `productId` / clustering key `timestamp`, or ClickHouse/Timescale for rollups. Given up: joins, cross-product transactions, strong consistency — all of which NFR3 already conceded. The justification was sitting in his own NFRs, unspent.

**The notify-path hot key, missed entirely.** A viral product has a million subscribers; the consumer reads *all* subscriptions for that productId and evaluates each. A senior indexes by `(productId, targetPrice)` — a Redis sorted set scored by targetPrice, or a clustered index — and on a new price `p` range-queries `targetPrice >= p`. A drop from $500 to $499 wakes the twenty people who wanted $499, not the million who wanted $300. Same "shrink the candidate set instead of filtering a large one" move as his DSA round the same morning.

**Scheduling.** "A fleet of cron jobs with products assigned to instances" is the mid-level answer; the senior question is what happens when instance 7 dies holding 3M assignments. Static assignment means those products silently stale for the length of the failure-detection window — and the 99.99% *detection-availability* NFR is precisely what breaks. The shape that survives: a work queue keyed by `nextPollAt` (Redis sorted set or partitioned scheduler), workers lease with a visibility timeout, a dead worker's leases expire and get re-taken. It buys adaptive polling for free — volatile products short interval, a product static for six months backed off to daily — dropping the 167k/s fetch rate by an order of magnitude without touching freshness on anything anyone cares about.

**Operability, none of which came up.** The metric that matters isn't QPS, it's **staleness**: max age of last successful poll, per product and per retailer domain. That's how you learn the system is silently wrong rather than loudly down — a retailer changes page markup, the parser returns nothing, prices freeze, every dashboard stays green. Also: consumer lag alerting tied to the 5-min NFR, a DLQ for unparseable pages with per-domain failure-rate alerts, and the cost of a fleet making 167k egress HTTP requests/sec — a real line item at that volume, and the strongest argument for adaptive polling.

### 4. Checklist
`system_design_senior_guidance.md` pre-round self-check. Two lines to read before the next round: *"Did I state avg and peak numbers, with the arithmetic, unprompted?"* and *"Did I push scale until something broke, and design for the broken case?"* Both were no today, and they're the two that produced the rating.
