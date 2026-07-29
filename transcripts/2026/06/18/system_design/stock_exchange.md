# System Design Round Transcript
**Date:** 2026-06-18
**Start Time:** 12:16
**End Time:** 13:20
**Duration:** 64 minutes
**Problem:** Design a Stock Exchange (Order Matching System)

---

## Conversation Log

**Interviewer:** Introduced the problem (exchange like NASDAQ / crypto: place orders, match, execute, publish market data). Asked for requirements.
**Aayush:** Asked whether traders can see live price for a stock.
**Interviewer:** Confirmed scope (place/cancel limit+market orders, match/execute, live market data). Asked him to commit FRs and give NFRs with numbers.
**Aayush:** Asked if 100M DAU x 5-10 orders/day is a fine assumption.
**Interviewer:** Accepted; pushed him to derive avg/peak orders/sec and latency/consistency.
**Aayush:** Gave FRs (place/cancel limit+market, match, view order status, view market data; out of scope: historical data, analysis tools) and NFRs (99.99% HA, p99<200ms matching, strong consistency / no double-sell, no double-charge, near real-time prices, 5k avg / 10k peak).
**Interviewer:** Flagged the throughput math: 100M x 10 = 1B/day; 5k looks low; challenged 2x peak multiplier for an exchange.
**Aayush:** Corrected: 10k orders/s avg (~1B / 10^5 s), 50k/s peak (5x).
**Interviewer:** Accepted. Rendered FRs/NFRs. Asked for core entities.
**Aayush:** User, Order(id,userId,side,stockSymbol,orderType,limitPrice), Trade(id,sellOrderId,buyOrderId), StockSymbol.
**Interviewer:** Probed missing quantity, lifecycle/partial fill, and tie-break field.
**Aayush:** Added Order.{createdAt, quantity, status, fulfilledQuantity}; Trade.{quantity, createdAt}.
**Interviewer:** Noted Trade needs execution price. Rendered entities. Asked for API (verbs, paths, req/resp, idempotency, near-real-time market data).
**Aayush:** JWT identity. POST /orders (idempotency-key header). GET /orders?cursor&limit (cursor pagination, dynamic inserts). GET /orders/{id}. GET /stockSymbols?symbol&cursor&limit.
**Interviewer:** Praised contract. Flagged missing cancel + that GET market data is polling, not real-time.
**Aayush:** DELETE /orders/{id}; upgrade /stockSymbols to WebSocket for bidirectional comm.
**Interviewer:** Accepted. Asked for WS command detail and then HLD write path.
**Aayush:** WS commands: subscribe(stockSymbol) client->server; pushMarketPriceUpdate(stockSymbol, currentPrice) server->client.
**Interviewer:** Asked for the write-path HLD end to end.
**Aayush:** Client -> API Gateway (auth) -> Order Service -> Kafka (partitioned by stockSymbol for per-symbol ordering, decoupling, durability; idempotency key dedup under at-least-once) -> Consumers pull and write order to Cassandra (chosen for write throughput / WAL; partition by stockSymbol, sort key createdAt). Self-raised hot-partition for popular stocks -> composite stockSymbol+timestamp partition, with scatter-gather read trade-off. Separate Matching Service matches buy/sell per symbol and records trades.
**Interviewer:** Rendered HLD (left Matching Service unconnected, deliberately). Asked: where does matching read from; where does the order book live; and how does Cassandra (AP) satisfy strong consistency / no double-sell.
**Aayush:** Matching reads buy + lowest active sell (market) or sell where price<=limit (limit); sort by createdAt+price. Cassandra has no locking and querying it per match bottlenecks at ~25k/s. Fix: consumer also writes order to in-memory Redis; keys stockSymbol_buy / stockSymbol_sell as sorted sets by price; Redis cluster, time-bucket to avoid hot keys; matching does atomic claim via Lua (no double-booking); writes Trade + updates status in DB. Redis crash failure -> persist to disk; reject orders while down.
**Interviewer:** Pushed scale-break: AAPL = 30% volume (~15k/s) at open. (1) time-bucketing the book across shards breaks atomic price-time matching; (2) keeping book whole bottlenecks one shard. Which, and how to resolve?
**Aayush:** Better not to shard the book — retain atomic price-time matching.
**Interviewer:** Asked: is 15k/s a real bottleneck for in-memory matching; how to fail over one engine per symbol without losing/reordering the book (Kafka helps?); and unit of sharding for the whole exchange.
**Aayush:** (1) One machine should handle it. (2) Kafka replay. (3) [pending]
**Interviewer:** Pushed for mechanism: replay from offset 0 is millions of msgs; how to bound recovery; how to avoid re-emitting duplicate trades on replay; unit of sharding/routing.
**Aayush:** Unit of sharding = stock symbol. Dedup: if order status is fulfilled in DB, skip matching it.
**Interviewer:** Noted per-order DB check is slow; replay from offset 0 still blows availability. What bounds replay depth?
**Aayush:** Snapshot the order book periodically and replay from last snapshot.
**Interviewer:** Confirmed. Rendered deep-dive. Closed round.

---

## Design Summary

**Requirements Gathered:**
- FRs: place/cancel orders (limit+market); match buy<->sell, execute trades; view order status; view market data (price, book depth). Out of scope: historical data, analysis tools.
- NFRs: 99.99% HA for order acceptance; p99 < 200ms matching; strong consistency (no double-sell); idempotent (no double-charge); near real-time market data. Scale: 100M DAU x 5-10/day = 1B/day; ~10k/s avg, ~50k/s peak (5x).

**High-Level Architecture:**
Client -> API Gateway (auth) -> Order Service -> Kafka (partition by stockSymbol) -> Consumers -> Cassandra (orders). Matching via in-memory Redis order book (sorted sets per symbol side), atomic claim via Lua, Trade + status written to Cassandra. Market data via WebSocket subscriptions.

**Key Design Decisions & Trade-offs:**
- Kafka partitioned by stockSymbol: per-symbol ordering + decoupling + durability; idempotency key dedups at-least-once.
- Cassandra for orders: high write throughput / WAL; composite stockSymbol+timestamp key to spread hot partitions (trade-off: scatter-gather reads).
- Redis in-memory book over matching-from-Cassandra: Cassandra can't lock and per-match queries bottleneck (~25k/s); Lua gives atomic claim.
- Book kept WHOLE per symbol (not time-bucketed) to preserve atomic price-time matching, accepting single-shard concentration for hot symbols.
- Unit of sharding = stock symbol.

**Scalability & Fault Tolerance Points:**
- In-memory matching fast enough for hot symbol (~15k/s AAPL) on one engine.
- Crash recovery: periodic order-book snapshot + Kafka replay from last snapshot offset.
- Replay dedup: skip orders already fulfilled in DB.
- Redis down: persist to disk, reject new orders temporarily.

**Gaps / Missed Areas:**
- Did not self-catch the Cassandra (AP) vs strong-consistency contradiction (needed prompting).
- Time-bucketing the live book (initially proposed) would break price-time matching.
- Dual-write consistency (Cassandra + Redis from consumer; trade write after Redis claim) not addressed.
- Exactly-once trade emission on replay not fully solved (per-order DB status check is weak).
- Market-data fanout path (engine execution -> WS subscribers) never connected; fanout/backpressure at 100M DAU untouched.
- No monitoring (consumer lag, stale book, latency SLA breach) and no cost discussion.
- Over time budget (64 min).

---

## Feedback Given

### Standard Evaluation
- **Requirements Clarification — 4/5** — Clean FR/NFR, self-raised consistency + idempotency; throughput math wrong on first pass, corrected with prompting.
- **Core Entities — 4.5/5** — Good once nudged; quantity/status/fulfilledQuantity/createdAt should have been volunteered.
- **API Design — 5/5** — JWT identity, idempotency key, cursor pagination justified, REST->WebSocket upgrade. Senior-level contract.
- **High-Level Architecture — 4.5/5** — Excellent write path; self-raised hot partition + composite-key trade-off.
- **Component Design & Trade-offs — 4/5** — Named alternatives well; missed the Cassandra/strong-consistency contradiction until challenged, then fixed it fully.
- **Scalability & Fault Tolerance — 4/5** — Reached in-memory book, snapshot+replay, Redis-down handling; made the correct whole-book call under the scale-break.
- **Deep Dive Quality — 4.5/5** — Strong naive->break->fix->new-failure->fix arc, mostly self-driven; answers got terse near the end.
- **Communication — 3.5/5** — Clear early; headline-only in deep dive; 64 min vs 45-50 target.

### Senior Readiness Debrief

**Senior-Signal Scorecard:**
- Own the narrative / self-raise traps — **Strong** (hot partitions, idempotency, consistency, Redis recovery mostly unprompted).
- Lead with trade-offs vs named alternatives — **Strong** (Cassandra vs locking, composite key vs scatter-gather, shard vs whole book).
- Push scale until it breaks — **Strong** (engaged AAPL-at-open break, correct whole-book call).
- API as a designed contract — **Strong** (idempotency, cursor pagination, REST->WS, JWT).
- Operability / second-order concerns — **Mixed** (got recovery/snapshot, Redis-down; missed dual-write consistency, exactly-once on replay, monitoring, cost).
- Pace — **Weak** (64 min vs 45-50).

**Overall level read:** Senior / hire, borderline strong-hire on technical axes. 4/6 signals Strong including scale-break and trade-offs (the L5 differentiators). Held back from clean strong-hire by pace and operability completeness, not design ability.

**What a senior strong-hire would have done on THIS problem:**
- Caught the Cassandra/strong-consistency contradiction unprompted; framed Cassandra as durable audit log, not matching store.
- Framed the matching engine as a single-writer-per-symbol sequencer consuming the ordered Kafka partition directly (the log IS the sequencer; matching is deterministic replay), making snapshot+replay and exactly-once natural.
- Owned the dual-write trap (order to Cassandra + Redis; trade to Cassandra after Redis claim can diverge); resolved via single source of truth = engine event output, with Cassandra/book as downstream projections.
- Closed the market-data loop: execution -> market-data service -> WS fanout; addressed fanout/backpressure at 100M DAU.
- Owned monitoring (consumer lag, stale book, p99 breach) and fleet cost without being asked.

**Self-drill:** system_design_senior_guidance.md items 5 (operability: dual-write, monitoring, cost) and 6 (pace).

**Time Taken: 64 minutes**
