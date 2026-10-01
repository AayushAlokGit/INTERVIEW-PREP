# System Design Round Transcript
**Date:** 2026-10-01
**Start Time:** 14:39:46 · **End Time:** 15:35:22 · **Duration:** 56 min
**Problem:** Robinhood (retail stock trading app)
**Difficulty:** 5/5 (Hard) — several independent hard problems that cannot all fit 45 minutes: order placement against an external exchange with no status-query API (dual write, timeout, idempotency, reconciliation from the trade feed only); a 1B messages/s price fan-out with a 5M-watcher hot symbol; cancel racing a fill; and a 10× market-open spike on both paths at once
**Dominant pattern:** realtime updates (price fan-out) + multi-step processes (order lifecycle against an external system), with a contention element on cancel-vs-fill
**Performance Rating:** 2/5  <!-- machine-read on future rounds; ≤2 = eligible for re-ask, ≥3 retired -->

**Would it have fit a real 45-min round?** No — cut off at the end of the high-level design; the deep dive would never have started

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 19 min (14:58:17) | +11 min | Over by 11 min |
| Core entities | 12 min | 25 min (15:04:21) | +13 min | Over by 13 min |
| API design | 17 min | 32 min (15:11:52) | +15 min | Over by 15 min |
| High-level design | 27 min | 45 min (15:24:26) | +18 min | Over by 18 min |
| Deep dive | 40 min | abandoned at 56 min (15:35:22) after one answer | — | Not completed |
| Wrap-up | 45 min | not reached | — | Not reached |
| **Total** | 45 min | 56 min (abandoned) | +11 min | Over, and incomplete |

## Round-trip Tax
| Phase | Parts asked | Answered 1st pass | Follow-up exchanges | Minutes lost | What was missing |
|---|---|---|---|---|---|
| Requirements | 2 (delivery fan-out number; measurable order guarantee + latency) | 0 | 2 | ~4 | Stopped at 100k/s inbound; first fan-out formula was 100k × 5M × 20; order guarantee contradicted the clarification he had been given |
| Core entities | 1 (does Order support FR2–FR4) | 0 | 2 | ~2 | CANCELLED status, then buy/sell; `id` and `userId` never added |
| API design | 2 (how a price / order status reaches the phone; 250 ms timeout case) | 0 | 2 | ~3 | Price delivery added (SSE) on prompt; order-status push never given; timeout case deferred |
| High-level design | 1 (rest of the design + two unnamed parts) | — | 1 | ~1 | First pass covered the price path only; order DB type, two component names and the GET handler left unstated |
| Deep dive | 5 | 0 | 0 | — | Round ended instead of answering |
| **Total** | 11 | 0 | 7 | ~10 of 56 min (18%) | |

**Deferrals used:** 1 — the 250 ms exchange-timeout case on `POST /orders`. Not returned to: he ended the round in the turn it was called in.

---

## Conversation Log
**Interviewer:** `[14:39:46 · +0m]` Design Robinhood, a retail stock trading app. Users watch live prices and place buy/sell orders. Robinhood is a brokerage: it routes orders to an external exchange. Difficulty: 5/5 (Hard). Reference timeline stated, measured not enforced. Start by gathering requirements.

**Aayush:** Questions -> 1. Do users get live updates for subscribed stocks? 2. How is the order state lifecycle managed, does the exchange expose APIs to query the order status? 3. Do we need to notify users on order status changes?

**Interviewer:** `[14:42:01 · +2m]` 1. Yes, live prices for stocks on screen. 2. The exchange gives a synchronous order API (place, cancel; returns exchange order id and accepted/rejected), a pushed trade feed (status changes and fills), and a pushed market data feed. No API to query order status on demand. 3. Yes, status should change in the app without refreshing.

**Aayush:** can the order be cancelled only until it has not been filled?

**Interviewer:** `[14:44:24 · +5m]` Yes, only while open. A partially filled order can have its unfilled remainder cancelled; a fully filled order cannot.

**Aayush:** FRs -> 1. Users can watch live prices of stocks to which they subscribe (exchange exposes a market data feed for every symbol). 2. Users can place orders for stocks they want to buy/sell, the brokerage routes orders to exchange. 3. Users can cancel placed orders until they are open (exchange streams order updates for orders placed by brokerage). 4. Users should be able to see their order status. Out of scope -> 1. Payment system.

**Interviewer:** `[14:46:26 · +7m]` FRs noted, payments out of scope. Go on to the NFRs.

**Aayush:** NFRs -> 1. Highly available system for live price updates. 2. Strong consistency for orders. 3. Price point update latency p99 < 1s. What scale should the system handle?

**Interviewer:** `[14:48:26 · +9m]` Givens: 20M DAU · 5 orders/user/day · 10,000 symbols · market data feed averages 10 updates/s per symbol · ~20 symbols on screen per user · 25% of DAU online at market open · peak 10× average · hottest symbol watched by 5M users at once · orders retained 7 years. Day = 10^5 s.

**Aayush:** 4. Support 20M DAU, 5 orders/user/day -> 1000 orders/s write load avg, peak 10x. 5. 10 price updates/symbol/s and 10,000 symbols so total price point update load is 100k price updates/s. 6. For a hot symbol there can be 5M subscribers so high fanout for price update.

**Interviewer:** `[14:53:56 · +14m]` Those are right. (1) Is 100k/s the number that sizes price delivery to users? If not, derive the one that does. (2) Not blocking: "strong consistency" and "highly available" have no measurable form — what must never happen to an order, and is there a latency target on placing one?

**Aayush:** 1. at peak time at market open 5M users are active each watching 20 symbols, so total price updates to be delivered is 100k*5M*20/s. 2. Order must never be cancelled if it has been partially filled. The latency target for placing order p99 < 250ms.

**Interviewer:** `[14:56:47 · +17m]` (1) Evaluate that product and sanity-check it: how many updates per second does one phone receive? (2) I said a partially filled order can have its remainder cancelled, so that rule doesn't match the product. 250 ms noted.

**Aayush:** 1. One user's phone should receive 200 price updates/s, so 5M should receive 1B price updates/s. 2. order when completely filled can't be cancelled.

**Interviewer:** `[14:58:17 · +19m]` Noted. FRs and NFRs on the canvas. Move on to core entities.

**Aayush:** Core Entities -> 1. User(id, name, email) 2. UserSymbolSubscription(userId, createdAt, symbolId, lastPriceUpdate) 3. PricePoint(symbolId, time, price) 5. StockSymbol(id, name) 6. UserPortfolioItem(userId, stockSymbol, quantityOwned) 7. Order(symbol, createdAt, quantity, status: OPEN | PARTIALLY_FILLED | FILLED, quantityFilled, exchangeOrderId)

**Interviewer:** `[15:02:23 · +23m]` Structural check: walk `Order` against FR2, FR3 and FR4. Does it hold everything each needs?

**Aayush:** needs cancelled status as well

**Interviewer:** `[15:03:03 · +23m]` That covers FR3. What about FR2 and FR4?

**Aayush:** buy / sell type is missing in order

**Interviewer:** `[15:04:21 · +25m]` Added. Entities on the canvas. Move on to API design.

**Aayush:** API Design -> identity from auth header. PUT /subscribe {stockSymbol} -> UserSymbolSubscription. POST /orders, Idempotency-key header, {symbol, quantity, type: buy/sell} -> Order(id, status: OPEN..). DELETE /orders/:id -> 2xx on successful cancellation of unfilled order; appropriate messages for partially filled and completely filled orders. GET /orders/:id -> Order(id, status). GET /orders?cursor={}&limit={} -> {Order[], nextCursor}.

**Interviewer:** `[15:09:02 · +29m]` (1) Structural: you sized 200 updates/s per phone under 1 s p99; PUT /subscribe returns a row. Where does a price reach the phone? Same for order status. (2) Not blocking: POST /orders returns OPEN — what does the client get if the exchange hasn't answered inside 250 ms?

**Aayush:** 1. GET /symbol will open an SSE connection through which the server will stream the price point updates as they arrive to the client.

**Interviewer:** `[15:10:31 · +30m]` (1) One connection per symbol or one per user? (2) You answered for prices only; how does an order status change reach the phone? The 250 ms question is still open; defer it if you prefer.

**Aayush:** 2. defer it. 1. one connection and price point updates for all the subscribed symbols streamed over that connection; the client makes one connection at startup with all the subscribed symbols and when a new symbol is subscribed it is added to the list.

**Interviewer:** `[15:11:52 · +32m]` Noted; deferral accepted. API on the canvas. Move on to the high-level design.

**Aayush:** The client will fire the GET symbols SSE endpoint on first start of app. The request is handled by the price point update streaming service, whose instance subscribes to a Redis pub-sub channel for the price updates for the relevant symbols. Another component listens to the price point update stream from the exchange and writes the price point update to a DB like Cassandra, which supports high write throughput; we don't need transaction guarantees so we can skip a SQL DB for price point data. Once written to the DB, updates are pushed to the Redis pub-sub channels for the relevant symbols so the streaming service can send them to the relevant users. Redis pub-sub is at-most-once so the client will still have to run a polling mechanism in case a price point update was missed.

**Interviewer:** `[15:18:10 · +38m]` Price path drawn. (1) The exchange listener has no name. (2) Which service and endpoint does the client poll? That covers FR1 only — describe placing, cancelling and seeing the status of an order.

**Aayush:** 1. Any name, give it whatever you think is relevant. 2. It can query the streaming service GET /symbol/:id/price?since={} to get the price points after a particular time from the DB. To place an order the client makes a POST handled by the order service; it performs necessary validations on the quantity of the symbol, places an order to the exchange and saves the order along with the exchange order id in the DB, and returns the order object. Another component watches the trade feed from the exchange, queries the order DB for the relevant exchangeOrderId and updates the order statuses. Users keep hitting GET /orders/:id to get the latest status; the client polls at regular intervals. To cancel, the client hits DELETE and the order service triggers the exchange's cancel endpoint; if rejected we relay the message to the client, else if accepted we relay the same.

**Interviewer:** `[15:24:26 · +45m]` Full design on the canvas. Unstated: order DB type, trade-feed component name, which service handles GET /orders/:id. Deep dive: the design has to meet 10,000 orders/s with orders never wrong, and 1B deliveries/s under 1 s p99. Where does it break first, and which part do you want to go deep on?

**Aayush:** 1. Order DB can be a SQL DB because we will query it very frequently and would benefit from ACID properties and indices. 2. you decide a name. 3. Order service will handle it. The load of 1B price deliveries has been largely reduced since users join different instances of the streaming service and these instances subscribe to a Redis pub-sub channel for a stock symbol. The price point update is pushed to the channel by a different component, which is effectively just pushing 100k price updates/s to a Redis pub-sub cluster; we could use the stock symbol as the hash key to distribute the data. Users connect to multiple instances in batches so each price point update is delivered to a few service instances which then relay it to the users connected. Regarding incoming order updates, there would be 10k orders placed in any second so 10k order updates coming in any second from the exchange; a SQL DB like PostgreSQL should be able to handle that write load.

**Interviewer:** `[15:31:46 · +52m]` Canvas updated. Prices: (1) how many streaming instances hold 5M connections and how many messages/s does one write out? (2) the hottest symbol has 5M watchers — how many instances receive each update and what does each do? (3) is 1B/s still leaving the system; if so what runs out first, if not what removed it? Orders: (4) the exchange accepts and the Order Service crashes before the DB write — what does the system know, and what does the Trade Feed Processor do when the fill arrives? (5) the deferred question: exchange hasn't answered in 250 ms — what does the client get, and what happens on a retry with the same Idempotency-Key?

**Aayush:** i am done with this round

**Interviewer:** `[15:35:22 · +56m]` Round ended. Feedback below.

---

## Design Summary
**Requirements Gathered:** Watch live prices for subscribed symbols; place buy/sell orders routed to an external exchange; cancel while open; see order status. Payments out of scope. NFRs: HA for prices, strong consistency for orders (a fully filled order can't be cancelled), price p99 < 1 s, order placement p99 < 250 ms, 1,000 orders/s avg and 10,000/s peak, 100k price updates/s inbound, 1B deliveries/s outbound at peak (200/s per phone), 5M-watcher hot symbol.

**High-Level Architecture:** Client → SSE (`GET /symbol`) → Price Point Update Streaming Service → subscribes to Redis pub/sub (channel per symbol). Exchange market data feed → Price Ingestion Service → Cassandra, then publish to Redis pub/sub. Client polls `GET /symbol/:id/price?since=` on the streaming service as a fallback. Client → Order Service → exchange order API, then save order + exchangeOrderId in PostgreSQL. Exchange trade feed → Trade Feed Processor → PostgreSQL (lookup by exchangeOrderId, update status). Client polls `GET /orders/:id`.

**Key Design Decisions & Trade-offs:** Cassandra over SQL for price points (write throughput, no transactions needed). SQL for orders (ACID, indices). Redis pub/sub accepted as at-most-once with a client polling fallback. Idempotency-Key on order placement. Cursor pagination on the order list. One SSE connection per client.

**Scalability & Fault Tolerance Points:** Symbol as hash key across a Redis cluster. Streaming service scaled horizontally. Nothing on failover, reconnect, reconciliation or monitoring.

**Gaps / Missed Areas:**
- Exchange called before the DB write: a crash in between leaves a live order at the exchange with no local record, and the Trade Feed Processor then gets a fill for an exchangeOrderId it cannot find. Not raised; not answered.
- Idempotency-Key declared in the API but never stored or used anywhere in the design.
- Exchange timeout on placement: deferred, never answered.
- Order status delivery is client polling, against the stated requirement of no refresh; no push channel.
- 1B/s fan-out declared "largely reduced" by pub/sub. It is not: every one of those messages still leaves a streaming instance. No instance count, no per-instance send rate, no conflation.
- Cassandra write sits in front of the publish, inside the 1 s latency budget.
- Polling fallback at 20 symbols per phone is itself a large read load on Cassandra; not sized.
- `Order` has no `id`, no `userId`, no order type or limit price, no PENDING or REJECTED state. No Fill entity. Nothing reserves shares or cash, so the same shares can be sold twice.
- Cancel treated as synchronous; the cancel-vs-fill race at the exchange was not addressed.
- Trade feed: no dedupe, ordering, or handling of a feed disconnect.
- Market-open spike (10× orders plus 5M connections at once) not addressed.
- No monitoring, no cost.

---

## Feedback Given

### Evaluation
- **Requirements — Mixed.** The three opening questions were the right ones: the second (does the exchange expose a status query?) is the question the whole order design turns on, and you asked it unprompted. You asked for scale and stated something out of scope. Against that: the NFRs started as three adjectives, the fan-out number had to be asked for, the first formula for it was 100k × 5M × 20, and the order guarantee you wrote contradicted an answer you had been given ten minutes earlier. 19 minutes against 8.
- **Core entities — Weak.** `Order` shipped without an id, an owner, a side or a cancelled state. Two probes recovered two of the four. No Fill, nothing that holds reserved quantity.
- **API design — Mixed.** Idempotency-Key on placement, cursor + nextCursor on the list, and the order id in the response were all there first pass. Missing: any way for a price to reach the phone until asked, any push for order status at all, order type and limit price, error semantics ("appropriate messages"), unsubscribe.
- **High-level architecture — Mixed.** Coherent, and yours — nothing in it was led out of you. Both paths have the right components. The order path has the dual-write hole at its centre.
- **Component design & trade-offs — Mixed, leaning weak.** Cassandra vs SQL for prices and SQL for orders each came with a reason. SSE, Redis pub/sub, polling for order status and exchange-first ordering came with none.
- **Scalability & fault tolerance — Weak.** "The 1B load has been largely reduced" is the claim the round needed tested and it was not sized. PostgreSQL at 20k writes/s was asserted. No failure handling anywhere.
- **Deep dive — Weak.** One answer, which restated the HLD, then the round was ended when the five concrete questions arrived.
- **Communication — Mixed.** Written answers were clear and well organised. Naming components was handed back twice ("you decide a name").
- **Diagram — matches what you said.** Data flow is directional and readable on both paths. The gaps in it are the design's gaps.

### Senior Readiness debrief

**0. Pace report**

| Phase | Reference | Actual | On pace? |
|---|---|---|---|
| Requirements | 8 min | 19 min | Over by 11 min |
| Core entities | 12 min | 25 min | Over by 13 min |
| API design | 17 min | 32 min | Over by 15 min |
| High-level design | 27 min | 45 min | Over by 18 min |
| Deep dive | 40 min | abandoned at 56 min | Not completed |
| Wrap-up | 45 min | not reached | — |

**Would this have fit a real 45-minute round? No.** The high-level design finished at minute 45 exactly. A real interviewer would have ended the round there, with a diagram on the board and no deep dive at all — and on this problem the deep dive is the interview. Nothing about order correctness or the fan-out would have been discussed.

The biggest time sink was requirements: 19 minutes, 11 over. Seven of those were your own clarifying questions and FRs, which is fine. The other twelve went on NFRs that arrived in three instalments. Every later phase inherited that deficit; entities, API and HLD each took close to their own reference duration (6, 7 and 13 minutes against 4, 5 and 10).

This was a 5/5 problem, and it is true that not all of it fits 45 minutes. But the overrun was process, not size: the problem's size only bites in the deep dive, and you never got there. A complete deep dive on one of the two paths was available if requirements had closed at 8.

Round-trip tax: 7 follow-up exchanges, about 10 of 56 minutes (18%). None of the 11 parts I asked was fully answered first pass. That is lower than recent rounds in minutes, mostly because the answers, when they came, were short.

Deferrals used: 1 (the exchange-timeout case). It was the right thing to defer. It was not returned to — the round ended in the turn it was called in.

**1. Senior-signal scorecard**

| Signal | Status | Reason |
|---|---|---|
| Owns the narrative | Mixed | Self-raised: the exchange status-query question, Idempotency-Key, pub/sub at-most-once with a fallback, out-of-scope. Not raised: the exchange-then-DB crash, the timeout, cancel-vs-fill, order status push. Ended the round when the deep dive got specific. |
| Leads with trade-offs | Mixed | Cassandra vs SQL, and SQL for orders, each justified. No alternative named for SSE, Redis pub/sub, polling, or the write ordering. |
| Pushes scale until it breaks | Weak | Declared the 1B/s break solved without sizing a single instance; asserted PostgreSQL would cope. |
| API as a designed contract | Mixed | Idempotency-Key, cursor, async-safe order id present. No order type or price, no order-status push, no error semantics. |
| Operability | Weak | Nothing raised: no reconciliation, feed disconnect, market-open handling, or metric. |
| Pace | Weak | Requirements 19 vs 8; HLD done at 45; deep dive never reached in a real round. |

Overall read: mid-level. The design is correct in outline and self-driven through the HLD, which is real progress on being led. It stops exactly where the senior signal starts. **No-hire at senior** on this round.

**2. Performance Rating: 2/5.** A coherent design with good instincts at the edges (the first clarifying question, the idempotency header), but the scale break was waved away rather than addressed, the order path has an unhandled correctness hole, and the deep dive was abandoned.

**3. What a senior strong-hire would have done on this problem**
- **Said the hard part out loud at minute 2.** You asked whether the exchange has a status query and were told no. The senior follow-through is immediate: "then the trade feed is my only source of truth after acceptance, so I must never have an order at the exchange that I have no record of — I write my own row first." That one sentence sets the order design.
- **Written the order before calling the exchange.** Insert as PENDING with a client order id, send that id to the exchange, update to SUBMITTED on the ack. A crash leaves a PENDING row to reconcile, not an orphan. Your flow does it the other way round and loses the order.
- **Made the Idempotency-Key do something.** It becomes the client order id: unique in the database and passed to the exchange, so a retry after a timeout returns the same order instead of placing a second one.
- **Answered the timeout as a state, not an error.** Return 202 with PENDING; the stream delivers the outcome.
- **Looked at 200 updates a second per phone and rejected it.** No one reads that. Keep the latest price per symbol and send one batched frame per second per device: 1B/s becomes 5M frames/s, about 50k/s on each of roughly 100 gateways. That is the scale break and its fix, and it was one sanity check away from the number you had already derived.
- **Taken the database write out of the price path.** Publish first, persist asynchronously.
- **Pushed order updates over the same connection** rather than polling, since you had told yourself in FR terms that the user should not refresh.
- **Raised market open unprompted** — it was in the givens twice — and named two metrics: age of the oldest pending order, and staleness per symbol.

**4.** Review the pre-round checklist in `system_design_senior_guidance.md`, in particular "Did I push scale until something broke" and "Did I raise the idempotency / consistency / failure traps before being asked".

**5. Drill prescription** — see the table below.

---

## Drill Prescription
| # | Gap targeted (signal / weakness) | Exercise (on which system) | Timebox | Pass criterion |
|---|---|---|---|---|
| 1 | Owns the narrative / dual write to an external system — the Order Service calls the exchange before saving, and the round ended on the crash question | *Failure walk on the Robinhood order path.* Write the order placement sequence as numbered steps. For a crash after each step, and for an exchange timeout, state: what the DB holds, what the exchange holds, what the client sees, and what repairs it. Say where the Idempotency-Key is stored and what a retry returns. | 15 min | Every step has all four answers; no state in which the exchange has an order the DB does not; a retry with the same key never creates a second exchange order; no prompting |
| 2 | Pushes scale until it breaks — "the 1B load has been largely reduced", with no instance sized | *Scale break on the Robinhood price fan-out.* Starting from 5M connections × 20 symbols × 10 ticks/s: size the gateway fleet (connections and messages/s per instance), state what the hot 5M-watcher symbol does to each instance, name the first thing that runs out, and give the fix in 3 bullets with the new messages/s figure. | 10 min | Instance count and per-instance send rate written with arithmetic; the fix reduces outbound volume by a stated factor; one sentence on what the user loses |
| 3 | Pace — requirements took 19 min against 8, NFRs arrived in three instalments | `/design-sprint` on a system with an external dependency (payment system or Ticketmaster): FRs, NFRs with every derived number including the outbound/fan-out one, entities, API. | 17 min | Requirements closed by minute 8 with avg and peak figures and one measurable consistency statement; every entity has an id and an owner; zero follow-ups needed on NFRs |
<!-- A later "## Drill Follow-up (<date>)" section scores each row against its pass criterion -->
