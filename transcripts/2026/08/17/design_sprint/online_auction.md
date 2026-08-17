# Design Sprint Transcript (front half, timeboxed)
**Date:** 2026-08-17
**Start Time:** 18:43:08 · **End Time:** 18:58:08
**Problem:** Online auction — sellers list items with a start price and end time, buyers bid, highest bid at close wins, watchers see the live high bid
**Difficulty:** 4/5 — a single contended row under strict serializability, a 10k-watchers-per-auction fanout, and a scheduled close that must fire exactly once
**Front-half readiness: 3/5**
**Front half complete inside 17:00: yes — 15:00, closed voluntarily with two minutes unused.** First time in the record. What was missing was missing by omission, not by clock: no traffic model, no read:write ratio, no storage growth, no durability posture, no response body on the bid endpoint, no mechanism for the live-price NFR.

| Phase | Pace target | Landed at | ± vs target | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 8:00 | 8:35 | +0:35 | 3 (FRs+OoS, scale question, NFRs) | 2/5 |
| Core entities | 12:00 | 10:27 | −1:33 | 1 | 2/5 |
| API design | 17:00 | 15:00 | −2:00 | 1 | 2/5 |

**Budget allocation:** 0:00–1:59 FRs + out of scope · 1:59–5:05 scale question · 5:05–8:35 NFRs · 8:35–10:27 entities (1:52) · 10:27–15:00 API (4:33). Two minutes of the box went unused. No phase overran meaningfully; the failure is coverage, not pace.

**First-pass completeness:** One message per phase, zero back-fill, zero revisions, two minutes unspent. **This inverts the standing diagnosis.** The previous three sittings were slow-but-complete; this one is fast-and-incomplete. He did not run out of time — he ran out of checklist, shipping a requirements phase missing four of the seven walk items with time still on the clock.

**Plausibility check:** Not performed, and this time it had teeth. NFR2 (strong consistency for the highest bid) and NFR3 (bid accept p99 < 10ms) are mutually unsatisfiable: a linearizable read-modify-write on a row 10k people are hammering, at 3k bids/s peak, includes lock wait plus replication ack plus network. 100ms p99 is the honest figure. He wrote a number a senior interviewer challenges on sight and had no derivation behind it. The check he skipped is one sentence: *"10k watchers × 1 Hz on one auction is 10k QPS from a single row — that can't be pulled from the database, it has to be pushed; and I can't promise 10ms p99 on a contended linearizable write."*

**Arithmetic:** none attempted. He asked for the givens, received them in full, and derived nothing — the same failure mode as the DSA round earlier the same day, where he asked for constraints, was told `budget ≤ 10^15`, and wrote `int`. **Twice in one day, on unrelated material: requests the constraint, then doesn't spend it.**

The numbers that were one line each and never appeared:
- `10M DAU × 10 bids / 10⁵` = **1,000 bids/s**, peak **3,000/s**
- `1M new auctions / 10⁵` = **10 listings/s**
- **10k concurrent watchers × 1 update/s = 10,000 QPS from a single auction row** — 10× global write throughput, from one key. The number that decides the design.
- `10⁸ bids/day × 365` = **3.7×10¹⁰ bid rows/year** (~3.7 TB) — bid history is the storage break, auctions are not.

## What he produced (verbatim, as it stood at 17:00)

### Requirements
```
FRs ->
1. Sellers can list items for auction with a starting price and auction end time.
2. Buyers can see active auctions and join any live auction.
3. Buyers can place bids on auction item.
4. Highest bid at auction close time wins
5. Buyers in an auction can always see the current highest bid price.

Out of scope ->
1. Fraud detection.

NFRs ->
1. Highly available for accepting bids (99.99 ~ 52mins downtime/year)
2. Strong consistency for displaying highest bid price in auction.
3. Low latency bid accept (p99 < 10ms)
4. Low freshness latency for highest bid price in auction (p99<1s)
```
Clarifying questions asked: one — "what scale can I expect for the system?" (at 1:59, answered 5:05 with 10M DAU · 100k live auctions · 1M new auctions/day · 10 bids/buyer/day · 10k concurrent watchers on a hot auction · 1yr retention · peak 3×).

### Core entities
```
1. User (id, name)
2. Item (id, name, price)
3. Auction (id, startedby:userId, status: ACTIVE | ENDED, endTime, startPrice, currentHighestBid)
4. Bid (auctionId, userId, placedAt, amount)
```

### API design
```
Identity from auth header

POST /auctions
Request: {itemId, startingPrice, endTime}
Response: Auction(id, startTime, endTime, startPrice, status: ACTIVE)

GET /auctions?cursor={}&limit={}
Response: Auction(id, status:ACTIVE, startPrice, endTime)[]
Cursor based pagination for dynamic lists

POST /auctions/:id/bids ->
Request: {amount}
Idempotency key header to dedup client side retries.

GET /auctions/:id
Response: Auction(id, startTime, endTime, currentHighestBid)
```

## What was still missing at 17:00

**Requirements:** the entire traffic model (no QPS of any kind) · read:write ratio · storage growth · durability posture — the one NFR an auction cannot hand-wave, since a lost bid is a legal dispute · consistency stated for only one subsystem · a latency target that contradicts the consistency requirement above it · out-of-scope list of a single item (no payments, escrow, shipping, proxy bidding, reserve price, cancellation, notifications, search ranking).

**Entities:** no `currentHighestBidderId` — the system cannot answer who is winning, which is FR4 · `Bid` has no id and no status, so "was my bid accepted or outbid?" is unanswerable **and the idempotency key on the bid endpoint has no key to deduplicate against** · nothing causes the `ACTIVE → ENDED` transition: FR4 needs a scheduled close and 100k live auctions make it a leased-job problem, not a field · nothing models a watcher, despite FR5 plus a sub-second freshness NFR plus a given of 10k concurrent watchers · `Item.price` alongside `Auction.startPrice` is either redundant or two names for one thing · no keys, no uniqueness constraints, `currentHighestBid` not flagged as denormalised.

**API:** **`POST /auctions/:id/bids` has no response body** — the most important response in the product; the client cannot learn whether the bid was accepted, whether it is now the high bidder, or what the price became · **FR5 has no delivery mechanism** — no WebSocket, no SSE, no long-poll, not even a stated polling contract, against a p99 < 1s freshness NFR · no `next_cursor` returned on `GET /auctions` despite cursor pagination being justified in the same line (identical to the Instagram likes-endpoint defect) · no bid-history endpoint · no "my bids" / "my auctions" · no cancel or retract, and no stated reason · no error semantics for the three obvious rejections (bid too low, bid after close, auction not found) · no page-size cap · no server time for countdown rendering.

## Where the time went
Nowhere unusual — every phase landed at or under target and two minutes went unused. 3:06 of the 8:35 requirements phase was the round-trip on the scale question, whose answer was then never used. The API phase took 4:33 and produced four endpoints with one request body between them. Nothing was re-derived, revised or narrated; the scaffold simply wasn't run to the end.

Worth preserving: cursor pagination justified in-line; an `Idempotency-Key` placed on the correct endpoint — the most contended write in the system; identity-from-auth stated once rather than repeated; `POST /auctions` returning the created object with its status; and the first front half in the record to close inside the box.

## Ideal front half (writable in the same 17 minutes)

### Requirements (target 8:00)
**FRs:** 1. Seller lists an item with start price and end time. 2. Browse/search active auctions. 3. Place a bid. 4. Highest bid at close wins; the auction transitions to ENDED exactly once. 5. Watchers see the current high bid within a second.
**Out of scope:** payments & escrow, shipping, fraud detection, proxy/auto-bidding, reserve prices, cancellation, seller ratings, search ranking, notifications.
**NFRs:**
- **Availability:** bid path 99.99% (52 min/yr) — a rejected bid near close is a lost sale; browse 99.9%.
- **Consistency per subsystem:** bid acceptance **strictly serializable per auction** (one winner, monotonically increasing price — the correctness core); catalogue/browse eventual, seconds stale; price broadcast eventual but bounded ≤ 1s.
- **Latency:** bid accept p50 30ms, **p99 100ms** — a linearizable write on a hot row cannot promise 10ms; price broadcast p99 < 1s; browse p99 < 200ms.
- **Traffic:** `10M × 10 / 10⁵` = **1,000 bids/s**, peak **3,000/s**. `1M / 10⁵` = **10 listings/s**. A hot auction has **10k concurrent watchers**; at 1 update/s that is **10,000 pushes/s from one row** — 10× global write throughput from a single key.
- **Read:write ≈ 100:1** globally, far worse on the hot key.
- **Storage:** `10⁸ bids/day × ~100B` = 10 GB/day = **3.7 TB/yr**, `3.7×10¹⁰` rows/yr; auctions 365M rows/yr. Bid history is the growth driver.
- **Durability:** an accepted bid survives node loss — synchronous replication before ack. A lost bid is a legal dispute, not a UX bug.
- **Plausibility check, out loud:** *"10k watchers × 1 Hz on one auction is 10k QPS from a single row — that cannot be pulled from the database, so the price must be pushed from a single writer. And I can't promise 10ms p99 on a linearizable contended write; 100ms is honest."*

> **What this buys:** the four absent walk items, plus the one number (10k pushes/s from one row) that turns FR5 from a feature bullet into the hardest problem in the system, plus a latency target that no longer contradicts the consistency requirement above it.

### Core entities (target 12:00)
```
User          (user_id PK, handle UNIQUE)
Item          (item_id PK, seller_id, title, description)
Auction       (auction_id PK, item_id, seller_id, start_price_minor, ends_at,
               status ENUM{ACTIVE,ENDED,SETTLED},
               high_bid_minor↓, high_bidder_id↓, bid_count↓, version)
Bid           (bid_id PK, auction_id, bidder_id, amount_minor, placed_at, idempotency_key)
               UNIQUE(auction_id, bidder_id, idempotency_key)
               UNIQUE(auction_id, amount_minor)
CloseJob      (auction_id PK, fire_at, claimed_by, claimed_until)   ← under load
Subscription  (auction_id, connection_id, server_id)                ← under load
```
`↓` = denormalised onto `Auction` so the read path is one row rather than an aggregate over 10⁸ bids.

**Entities that only appear under load:** **`CloseJob`** — 100k live auctions each need a fire-at-`ends_at` with a lease so exactly one worker closes each auction exactly once; FR4 is a scheduling problem, not a field. **`Subscription`** — forced by the 10k-watchers figure: pushing a price update requires knowing which connections on which servers care, a registry that does not exist at small scale where clients simply poll.

> **What this buys:** an owner for the winning bid (`high_bidder_id`), a `bid_id` for the idempotency key to deduplicate against, something that causes the `ENDED` transition, and something that knows where the live updates go.

### API design (target 17:00)
Identity from the auth header; `Idempotency-Key` required on every unsafe method.
```
POST   /v1/auctions              {item_id, start_price_minor, ends_at}
       → 201 {auction_id, status:"ACTIVE", ends_at}
GET    /v1/auctions?status=active&ending_before=&cursor=&limit≤50
       → 200 {items[AuctionSummary], next_cursor}
       cursor on (ends_at, auction_id) — offset skips rows as auctions close mid-page
GET    /v1/auctions/{id}
       → 200 {auction_id, status, high_bid_minor, high_bidder_handle, bid_count,
              ends_at, server_time, version}      server_time so the client's countdown
                                                  survives clock drift
POST   /v1/auctions/{id}/bids    {amount_minor, expected_high_bid_minor?}
       → 201 {bid_id, accepted:true, new_high_bid_minor, you_are_high:true, version}
       → 409 {accepted:false, reason:"OUTBID", current_high_bid_minor}
       → 410 {reason:"AUCTION_CLOSED"}
       → 200 replay of the original result when the Idempotency-Key repeats
GET    /v1/auctions/{id}/bids?cursor=&limit≤50   → {items[], next_cursor}
GET    /v1/users/me/bids?cursor=&limit≤50        → {items[], next_cursor}
WS     /v1/auctions/{id}/stream   → pushes {high_bid_minor, high_bidder_handle, version,
                                    ends_at} on every change; version is monotonic so a
                                    client discards reordered frames.
                                    Fallback: GET /v1/auctions/{id} polled at 1 Hz.
```
**Errors:** 400 validation · 401 · 403 seller bidding on own auction · 404 · 409 outbid / version conflict · 410 closed · 429 + `Retry-After` (bid spam on a hot auction) · 503 retry-safe on all idempotent methods.

> **What this buys:** the bid response — `accepted`, `you_are_high`, `current_high_bid_minor` — the field set the whole client UI branches on and which was entirely absent; a delivery mechanism for FR5 instead of an unbacked NFR; `next_cursor` actually returned on the list whose pagination he justified; and status codes for the three rejections every auction has.

## Feedback given
- **First front half in the record to close inside 17:00** — 15:00, voluntarily, two minutes unused. Keep that.
- But the standing diagnosis inverts: this was fast-and-incomplete, not slow-and-complete. Four of seven walk items were omitted with time still on the clock — that is a checklist failure, not a pace failure, and it has a different fix.
- Asked for the givens and derived nothing from them. Same failure as the morning's DSA round, where he asked for constraints, got `budget ≤ 10^15`, and wrote `int`. Twice in one day: requests the constraint, doesn't spend it.
- NFR2 and NFR3 contradict each other — 10ms p99 on a strongly-consistent contended write is not achievable, and no plausibility check was run on either.
- The bid endpoint returns nothing; FR5 has a freshness NFR and no mechanism; `next_cursor` was justified and then not returned. Third consecutive sprint where the load-bearing response field is the gap.
- Real credit: idempotency on the correct (most contended) write, cursor pagination justified in-line, identity-from-auth stated once, created object returned on `POST /auctions`.
- The one habit to change: **before leaving the requirements phase, write the QPS line — `DAU × actions ÷ 10⁵` — even if nothing else gets written.**

**Sitting note:** problem 2 was not served; the sitting was closed after problem 1 by request.
