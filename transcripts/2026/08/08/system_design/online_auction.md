# System Design Round Transcript
**Date:** 2026-08-08
**Start Time:** 14:44 · **End Time:** 15:51 · **Duration:** 67 min
**Problem:** Online Auction System
**Difficulty:** Medium
**Dominant pattern:** contention (with scaling writes)
**Performance Rating:** 3/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->

**Would it have fit a real 45-min round?** No — cut off at the end of the high-level design; the entire deep dive would never have happened.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 16 min | +8 | No |
| Core entities | 12 min | 20 min | +8 | No |
| API design | 17 min | 31 min | +14 | No |
| High-level design | 27 min | 45 min | +18 | No |
| Deep dive | 40 min | 66 min | +26 | No |
| Wrap-up | 45 min | 67 min | +22 | No |
| **Total** | 45 min | **67 min** | +22 | No |

---

## Conversation Log

**Interviewer:** Design an Online Auction System — eBay-style: sellers list items with a start price and an end time, buyers place bids, and when the auction closes the highest bidder wins. Difficulty: Medium. Start with requirements — FRs, what's explicitly out of scope, and NFRs with concrete numbers.

**Aayush:**
FRs ->
1. Sellers list items for auction with start price and end time for auction.
2. Buyers can participate in auctions.
3. Buyers place bids for item in an auction
4. At auction close time the buyer with highest bids wins the item.

**Interviewer:** Two things before we move on: what's explicitly out of scope? And NFRs with numbers — how many auctions live at once, bids/sec, read/write ratio, and where do you sit on consistency vs availability for each part?

**Aayush:** the current maximum bid price also needs to be displayed another FR

**Interviewer:** Noted — FR5. Now the other two: out of scope, and NFRs with numbers.

**Aayush:**
NFRs ->
1. Highly available for accepting buyer bids (99.99~52mins downtime/year)
2. Strong consistency for placed bids.
3. Lets assume system needs to support 10,000 concurrent auctions with 100 bidders in each auction. 10k bids/s avg peak 10x.

**Interviewer:** (1) You've asked for 99.99% availability and strong consistency in the same breath — under a partition you only get one. Which do you sacrifice, for which operation? (2) Where does 10k bids/sec come from — walk the arithmetic from 10,000 auctions x 100 bidders. Is peak diurnal or auction-specific? (3) Still open: read/write ratio, p99 latency for a bid, and out of scope.

**Aayush:**
read/write ration -> 100x many bids come in but only need to read the current max bid
strong consistency for bids is non negotiable.
latency target for bid to be accepted p99 < 150ms
Out of scope - Retaining all bid history for all auctions

**Interviewer:** (1) If strong consistency is non-negotiable you lose availability on the write path during a partition — is that the trade, and does it apply to reads? (2) The arithmetic: what bidding behaviour gets you from 1M bidders to 10k bids/sec?

**Aayush:**
1. th rationale for strong c0nsistency is that every bidder must see the highest bid placed so far , so a write should be visible to all readers.
2. there would be 1M bids/s and peak would be 10x

**Interviewer:** You've moved from 10k to 1M bids/sec — 100x apart. How often does a single active bidder place a bid? Derive the QPS from that rate. And I'm taking your consistency answer as: reads must reflect the latest committed bid, no stale current price — confirm.

**Aayush:** assuming a single activ user places 1 bid/s. yes the current maximum price should not be stale as that will dictate the future bids of the bidders

**Interviewer:** Locked in: 1M bids/sec average, 10M peak, reads never stale. Next: core entities — objects and their important fields, before any boxes and arrows.

**Aayush:**
Core Entities ->
1. User (id, name, email, ..)
2. Item (id, name , ..)
3. Auction (id, itemId, startingPrice, endTime, startTime, sellerId)
4. Bid (bidTime, price, buyerId, auctionId)

**Interviewer:** (1) FR5 — which entity answers "current maximum bid", and is it derived or stored? (2) FR4 — which entity records the outcome? Nothing here can answer "is this auction over, and who won?"

**Aayush:**
1. For now the maximum bid will be a field in the auctionEntity
2. The auction entity

**Interviewer:** Name the exact fields you're adding, then move into API design — verbs, paths, explicit request/response shapes with named fields, pagination where it applies, and how a write behaves under retry.

**Aayush:**
APIs ->
NOTE: Identity extracted form auth headers

POST /auctions
Request: {itemId , startPrice, endTime , startTime}
Response: Auction (id, startTime, startPrice, status, winnerId:null , winningPrice, currentMaxPrice;null)
Idempotency header to deduplicate client side retries.

POST /auctions/:id/bid
Request: {price}
Response: 2xx status indicating bid accepted
Idempotency key header to dedup client side retries.

GET /auctions/:id/maxPrice
Response: Auction(maxPrice)

**Interviewer:** (1) The common outcome in an auction is rejection — give me the response shape and status codes for both paths. (2) No list endpoint — how do buyers find auctions, and what's your pagination scheme and why? (3) What does the server actually do with the idempotency key on bid, where is it stored, how long retained — at 1M bids/sec that store isn't free.

**Aayush:**
1. I am accepting all bids , bids will be rejected only when auction has ended in this case return a status indicating auction ended.
2. GET /auctions?cursor={}&limit={}
Response: Auction(id, item[ItemName ], currentMaxPrice..), nextCursor:""
Cursor based pagination to allow auctions being dynamically inserted.
3. No need for idempotency key for the bid endpoint client side retires can place multiple similar bids which dont really matter much since only 1 bid wil l be selected anyways as the winner

**Interviewer:** (1) A $5 bid arrives while currentMaxPrice is $500 — you return 2xx and the bidder sees success but hasn't won and the price didn't move. Is that the semantics you want? (2) You've committed to writing every retry as a real Bid row at 1M bids/sec — do you accept (a) the write amplification, and (b) that "only one bid wins anyway" breaks if the duplicate arrives after the close instant?

**Aayush:**
1. accept all bids
2. The write amp-licfication will increase so need to have idempotency key and not store dulicate bids

**Interviewer:** Now the high-level design. Boxes, arrows, what each arrow carries, the bid write path and the currentMaxPrice read path, concrete systems named, and what you gave up versus the alternative.

**Aayush:** *(mid-turn)* also eventual consistency is acceptable for bids at any time just the knowledge of the maximum price is needed not all the bids

**Interviewer:** Meaningful revision noted — bids eventually consistent, only currentMaxPrice strongly consistent. Go ahead with the HLD.

**Aayush:** a buyer can place a bid in an auction the POST request will be handled by the bid service which will write the bids to the DB and also update the current max bid price in auction entity. A simple read path for maximum price would fetch the auction entity and return the current mx price.
Now in order to fit this simple design to our scale of high write we would need a DB like dynamo DB which would be able to support our high write throughput needed. The tradeoff is that it provides eventual consistency which is acceptable for bids in our case.
Also if the service will need to be horizontally scaled to handle the hgih write load. However with this design if the service crashes then the bids which it had accepted and not yet written are not lost. So we can introduce a kafka queue pairitoned by auctionId , this would ensure that the bids are durably stored in kafka and the bids of an auction are naturally ordered.
There will be idemptent consumers which will write the bids to the DB. The DB in turn will also be paritioned by auctionId , and the bidTIme would be used as the sortKey.
Now for the read path , if we allow the read requests to fall on the DB then the DB will be overloaded with write and read requests so we can remove the DB from the hot rea path by storing the currentMaxBidPrice in a redis cache. The read path will query the cache for the max price. Thsi will reduce load on DB and also reduce latency of read path and allow for higher read throughput. However here the redis cache will need to be kept consistent. This can be done by havinf CDC mechanism to keep the current max price in redis upto date.
If redis fails then read pressure falls back on the DB. In this case of redis failure we can use request coalescing to ensure DB faces only 1 read request.
FOr observability we can have a DLQ in kafka where bids which were not written to the DB would be stored for human investigation.
We woudl also need ot mintor the queue depth and scale consumers upon increasing queue depth to keep bid accept latency down , and prevent backpressure.

**Interviewer:** Clarifying: who writes currentMaxPrice — the consumer or the Bid Service? Is Redis updated only via CDC? Deep dive: you said the max price must not be stale, but the write path is Bid Service -> Kafka -> consumer -> DynamoDB -> CDC -> Redis, four hops. In the last 10 seconds of a hot auction a bidder reads $500 and bids $510 while bids of $600/$650/$700 are in flight. Walk me through what your system does.

**Aayush:**
the idempotent consumer writes the currentMaxPrice.
Updating redis via cdc will intorudce delays instead we can update redis directly from the bid service itself.

**Interviewer:** That creates three new problems. (1) Two Bid Service instances write the same Redis key — $700 then $510 — what is currentMaxPrice after, and what exactly do you write so this can't happen? (2) Redis synchronous, DynamoDB async — they diverge. Which is authoritative at close, and how do you know it's right? (3) A Redis outage was a load problem before; now the fresh price only exists there. What does node loss cost you mid-auction?

**Aayush:**
1. use lua atomic scripts and always the maximum value is needed .
2. The dynamo DB store is authoritative because we can verify it aginst the actual bids
3. The bid service can fail and stiop accepting bids

**Interviewer:** Now push the scale. One auction goes viral — 500,000 bids/sec on a single auctionId. Everything keys on auctionId: one Kafka partition, one DynamoDB partition key, one Redis key with a synchronous Lua CAS per bid. Give me the numbers: what does one Kafka partition sustain, what does one Redis node sustain, where does 500k/sec put you? Then how do you redesign, and what do you give up?

**Aayush:** we would have to dynamically minotr the incoming bid rate for each auction and when we notice the bids crossing a threshold then we can split the bids into multiple kafka shards , and similarly split the write load acorss myltiple shards in the DB.
The load can also be split across multiple redis nodes.
This introduces complexity at read time because now scatter gather queries are necessary.
Not sure of the numbers

**Interviewer:** Attempt the numbers — Redis ~100k ops/sec/node, Kafka partition ~10 MB/s, ~200 bytes/bid. How many Redis sub-shards, how many partitions? And with the max split across N sub-keys, reads scatter-gather N ways — at 100:1 does your fix help the read path or move the problem?

**Aayush:** no read to write ration is 1:100 the write side skew is heavy

**Interviewer:** Fair — writes are 100x reads, that's what you said; the read fan-out is cheap and your sharding fix stands. Still owe me the numbers. Then FR4 — closing the auction. Who notices endTime, who picks the winner, and how do you decide which bids are in, given a bid can sit in Kafka before the consumer writes it?

**Aayush:**
5 redis nodes for 1 auction.
10 kafka paritions per auction
A cron job runs the winner calculation which is schedule dto run after some grace period of time to allow for any remiaing bids to come in

**Interviewer:** Both numbers check out. (1) With 10,000 auctions ending at arbitrary times, what does the cron scan and how often — cost, and worst-case delay from endTime to winner? (2) The grace period is a guess — what signal tells you it's safe? You already have a metric that answers that. (3) A bid arrives at endTime - 1ms but the auction closes before the consumer processes it — in or out, and which clock decides?

**Aayush:**
1. When the auction is created then a cron can be scheduled to execute after a endTime + grace period.
The cron will scan all the bids for an auction and determine the highes bid among them. For this the cron will hav to scan all the bids of the auction
2. The queue depth of the paritin associated with the auction if there is non zero depth => some bids remain.
3. The idemptent consumer will check the bidTime and the nedTime before making a decision , so even if processing takes time if the bid should be allowed then it will.

**Interviewer:** Consumer lag as the safety signal is the right answer and you found it yourself. Final two: (1) Whose clock stamps bidTime? (2) The cron scans all bids — but you already maintain Auction.currentMaxPrice and you scoped bid history out. Do you need the scan? Then: biggest remaining bottleneck and the trade-off you'd accept.

**Aayush:**
1. Best to have the API gateway clock stamp it.
2. The auction entity can be kept upto date with the maximum bid as bids come in and just using the auction entity will be enough to decide the winner

**Interviewer:** Last thing: the single biggest remaining bottleneck, and the trade-off you'd accept to fix it.

**Aayush:** not sure what bottleck remains ?

**Interviewer:** That's the round.

---

## Design Summary

**Requirements Gathered:**
- FRs: seller lists item (start price, end time); buyers participate; buyers place bids; highest bid at close wins; current max bid displayed.
- Out of scope: retaining all bid history for all auctions.
- NFRs: 99.99% availability for accepting bids; eventual consistency acceptable for bids, strong consistency required for currentMaxPrice (revised mid-round from "strong consistency for bids"); 10,000 concurrent auctions x 100 bidders = 1M active bidders at 1 bid/sec = 1M bids/sec avg, 10M peak; write/read ratio 100:1; bid accept p99 < 150 ms.

**High-Level Architecture:**
Buyer -> Bid Service (horizontally scaled) -> Kafka (partitioned by auctionId, durable, natural per-auction ordering) -> idempotent consumers -> DynamoDB (PK auctionId, SK bidTime; stores Bid rows and Auction.currentMaxPrice; authoritative). Bid Service also writes currentMaxPrice directly to Redis on the synchronous path using an atomic Lua script that keeps the maximum. Read path queries Redis; on Redis failure, falls back to DynamoDB with request coalescing. Failed consumer writes go to a Kafka DLQ for human investigation. Queue depth is monitored and consumers autoscale to prevent backpressure. A cron scheduled at auction creation for endTime + grace period determines the winner from Auction.currentMaxPrice.

**Key Design Decisions & Trade-offs:**
- DynamoDB for write throughput; accepted eventual consistency for bid rows.
- Kafka introduced for crash durability of accepted-but-unwritten bids, and for natural per-auction ordering.
- CDC (DynamoDB -> Redis) rejected in favour of a direct synchronous Redis write, trading a second writer / divergence risk for freshness.
- DynamoDB kept authoritative because it can be reconciled against the actual bid rows.
- Hot auction: dynamic per-auction bid-rate monitoring, sharding across multiple Kafka partitions / DB shards / Redis nodes above a threshold; accepted scatter-gather on reads as the cost (cheap given 100:1 write skew).
- Idempotency key on bid, adopted mid-round specifically to avoid write amplification from retries.
- Winner computed from the maintained Auction.currentMaxPrice rather than a full bid scan.

**Scalability & Fault Tolerance Points:**
- Sizing under a 500k bids/sec hot auction: 5 Redis nodes (500k / 100k per node), 10 Kafka partitions (500k x 200B = 100 MB/s / 10 MB/s per partition). Both correct.
- Kafka DLQ for poison/failed writes.
- Queue-depth-driven consumer autoscaling for backpressure.
- Request coalescing on cache failure to protect the DB.
- Consumer lag on the auction's partition used as the correctness signal for when it is safe to compute a winner.
- Server-side (API gateway) clock stamping of bidTime; consumer compares bidTime to endTime to resolve the close boundary race.

**Gaps / Missed Areas:**
- Never self-raised: the staleness contradiction against his own NFR, the hot-partition problem implied by his own partition key, concurrent writers on the Redis key, the auction close mechanism, the clock question.
- Never named a rejected alternative for any component (DynamoDB vs Cassandra / sharded Postgres; Kafka vs Kinesis).
- Redis is on the synchronous path with no persistence, replication or failover story; "the bid service can fail and stop accepting bids" silently abandons the 99.99% availability NFR and was never reconciled.
- Bid response shape left as bare "2xx"; "accept all bids" semantics asserted twice, never defended.
- Traffic model (1M bidders x 1 bid/sec sustained) never sanity-checked; flipped from 10k to 1M bids/sec without flagging it.
- Punted on sizing numbers before attempting; produced correct numbers only after being pushed.
- Outbid notification and auction sniping / soft-close extension never surfaced.
- Could not name a remaining bottleneck at wrap-up.

---

## Feedback Given

### 0. Pace report

Requirements 16 min (ref 8), entities 20 (12), API 31 (17), HLD 45 (27), deep dive 66 (40), wrap 67 (45). Total 67 vs 45.

**Would this have fit a real 45-minute round? No.** The HLD landed at exactly the 45-minute mark, so in a real round the entire deep dive never happens — and the deep dive is where this round's best work was: hot-auction sharding, the Redis Lua CAS, consumer lag as the close-safety signal, both sizing calculations. All of it invisible. He'd be assessed on requirements + entities + API + a bare HLD, which is a worse assessment than he deserves.

**Biggest time sink:** requirements and API jointly — 31 minutes to produce four endpoints. The specific pattern is one question answered per turn: asked for out-of-scope + NFRs together, got FRs; asked for ratio + latency + scope, got two of three. Each round-trip costs 2-4 minutes and he needed roughly twice as many as a senior would. Second sink: API design taking three rounds of probing to extract shapes, pagination and idempotency — all three of which he clearly knows, since he produced them correctly the moment they were asked for.

### 1. Senior-signal scorecard

- **Owns the narrative — Mixed** (up from Weak). Genuinely self-raised in the HLD: Kafka for crash durability, DLQ, backpressure/consumer autoscaling, request coalescing. But every trap that mattered — staleness contradiction, hot key, concurrent max-price writes, close mechanism, clock — had to be handed to him.
- **Leads with trade-offs — Mixed.** Real unprompted ones: DynamoDB's eventual consistency accepted deliberately, CDC dropped for latency, scatter-gather named as the cost of sharding. Missing: never named a rejected alternative. "A DB like DynamoDB" is a category, not a decision.
- **Pushes scale until it breaks — Mixed.** He never pushed it; the interviewer did. Once pushed he handled it well and correctly, with both sizing numbers right. Failure mode: said "not sure of the numbers" first and only attempted after insistence.
- **API as a designed contract — Mixed.** Best API round he's had: auth-header identity unprompted, idempotency keys unprompted, correct cursor pagination with the right justification. Held back by the bare `2xx` bid response and the undefended "accept all bids".
- **Operability — Strong.** The standout. DLQ, queue-depth-driven autoscaling, request coalescing under cache failure, consumer lag as the correctness signal for close, server-side clock stamping. Using lag as the safety signal rather than a fixed grace period is a genuine senior answer, reached in one.
- **Pace — Weak.** 67 minutes for a 45-minute problem, deep dive starting exactly when a real round ends.

**Overall: senior-leaning content, mid-level delivery.** The finished design is a solid senior design; the process that produced it was interviewer-extracted and 50% over budget. On a real loop: **hire, not strong hire** — and only if allowed to run over, which he wouldn't be.

### 2. Performance Rating: 3/5 — Pass

Real improvement: reconciled a contradictory NFR himself mid-round, produced a complete write path with resilience patterns unprompted, best operability answers to date. Capped at 3 because the pace would have truncated the round before any of that landed, and because the three hardest problems — staleness, hot key, close semantics — were all raised by the interviewer.

### 3. What a senior strong-hire would have done on this problem

- **Self-raised the contradiction in their own NFRs.** Asking for a never-stale max price and then drawing a four-hop async pipeline is a conflict a senior catches at the moment they draw the Kafka box: "this conflicts with my own freshness NFR, so the max price leaves the async path and is updated synchronously; only the bid record goes through Kafka." That split — synchronous authoritative price, asynchronous durable log — is the core insight of the problem, and owning it unprompted is most of the round.
- **Named the alternative every time.** "DynamoDB over sharded Postgres because I need >1M writes/sec and I'll give up transactional multi-row consistency, which I don't need since the price is a single atomic CAS."
- **Sanity-checked the traffic model.** 1M bidders x 1 bid/sec is not human behaviour — bidding is bursty and concentrated at the end. "Steady state is maybe 30k bids/sec; the interesting number is the sniping burst where one auction takes 500k/sec for ten seconds." He reached the right design concern through a wrong model.
- **Owned the hot-auction case unprompted.** The moment you partition by auctionId, the hot-partition problem exists. Name it in the same breath as the partition key.
- **Defended or fixed "accept all bids."** $5 against a $500 max returning 200 OK is a product bug. Either `409 { currentMaxPrice, minimumBid }` or `{ accepted: true, isHighestBid: false, currentMaxPrice }`. Either is defensible; a bare 2xx asserted twice is not.
- **Answered the last question.** Bottlenecks left on the table: Redis now sits on the synchronous path with no persistence or failover story, and "the bid service can fail and stop accepting bids" silently abandons the 99.99% NFR. Also never surfaced: outbid notification (without it, the read ratio is really a polling storm) and auction sniping / soft-close extension, the single most important real-world auction behaviour.
- **The one process fix:** when asked a three-part question, answer all three parts in one message. Nothing else about his process needs to change — the content is close to the bar; the turn count is what's killing him.

### 4. Checklist

Re-read `system_design_senior_guidance.md`, specifically section 1 (own the narrative), section 3 (push scale yourself), and section 6 (pace). The pre-round self-check at the bottom is exactly the list he'd have passed today had he walked it himself before starting.
