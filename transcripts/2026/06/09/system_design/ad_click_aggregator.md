# System Design Round Transcript
**Date:** 2026-06-09
**Start Time:** 9:58
**End Time:** 11:03
**Duration:** 65 minutes
**Problem:** Design an Ad Click Aggregator

---

## Conversation Log

**Interviewer:** Presented the problem — online ad platform, clicks generated across publisher sites, advertisers need near-real-time aggregated click metrics by dimension. Asked Aayush to gather requirements.

**Aayush:** Do we also need to show dashboards for the click data?

**Interviewer:** Yes — a query/dashboard side (clicks per campaign/ad over time ranges, broken down by dimensions); near-real-time desirable (~1 min). So a write path (ingest) and a read path (query).

**Aayush:** Do we need to keep all data or can we specify a retention period?

**Interviewer:** Raw events ~30 days for reconciliation/audit; aggregated rollups kept much longer (months–years). Asked him to commit to FRs and NFRs with self-derived numbers.

**Aayush:** 
FRs: (1) Record click events from many publisher sites; (2) Dashboard for click metrics per ad/campaign over time ranges, dimensions = publishing site, region; (3) Near-real-time capture; (4) Raw data 30 days, aggregated data 1 year.
NFRs: (1) HA for reception 99.99%; (2) click→dashboard latency <1 min; (3) no click lost; (4) dashboard query p99 <1s; (5) 1M clicks/s avg, 10M/s peak; 1KB/event → 1 GB/s avg, 10 GB/s peak, ~100 TB/day.

**Interviewer:** Sanity-checked: 1M/s × 86,400 ≈ 86B events/day ≈ 86 TB/day raw, ~2.6 PB / 30 days. Noted the durability (no loss) vs availability tension. Rendered FRs/NFRs. Asked for core entities.

**Aayush:** ClickEvent (id, timestamp, publishingSite, region).

**Interviewer:** Pointed out the entity has no reference to which ad/campaign was clicked (needed for FR2), and that the dashboard doesn't query raw rows — is there another entity for the read path?

**Aayush:** Added Advertisement entity (ClickEvent links to it) and an AggregatedMetric entity with a timePeriod column (hour/day), a timeRange attribute, and an aggregateType (count/min/max).

**Interviewer:** Locked entities, flagged the ClickEvent `id` generation/uniqueness for later. Asked for API design — write + read, explicit shapes, pagination, idempotency.

**Aayush (first cut):** POST /clickEvents {timestamp, site, region}, 2xx, idempotency key header. GET /clickEvents?aggregate_interval&aggregate_type&dimensions&start&end → [AggregatedMetrics].

**Interviewer:** Flagged: write missing adId; read response vague; large time-range result needs pagination.

**Aayush (revised):** POST /ads/:id/clickEvents (adId in path), idempotency key header. GET /clickEvents?...&cursor&limit → [AggregatedMetric(timePeriod, value)]; cursor-based pagination, cursor = last clickEventTimestamp + clickEventId, to allow dynamic inserts without breaking pagination.

**Interviewer:** Praised cursor reasoning; noted response row should also carry the dimension value (e.g. which region). Rendered API. Asked for the HLD write path.

**Aayush:** Write path — user clicks → POST hits Ingestion Service → immediately append event to Kafka partition (durability + at-least-once); no ordering needed so any partition. Consumer group of workers pulls events, uses idempotency key for idempotent insert of ClickEvent into DB. At heavy write scale, use NoSQL (DynamoDB) over SQL. Partition by adId (click + aggregate data for an ad on same shard), sort key on timestamp. Workers write the ClickEvent and also update AggregatedMetric records for current hour and day matching the click's dimensions. Update must be atomic so concurrent worker updates aren't lost → acquire distributed locks on the DB record (short TTL to handle worker crash), since DynamoDB lacks inherent locking.

**Interviewer:** Rendered the write path. Asked him to complete the read path, then said he'd dig into the write-path claims.

**Aayush:** Read path — client hits GET, service retrieves aggregated metrics for the requested interval/time period by querying the DB. Because the write path updates aggregates per event, read-time work is easy.

**Interviewer:** Rendered read path. Deep dive: at 1M–10M clicks/s, what does a distributed lock per click cost in throughput/latency, and is a lock even needed for an increment given what DynamoDB offers?

**Aayush:** (1) Latency rises but an in-memory Redis lock store keeps it small; throughput drops. (2) To avoid throughput loss, use optimistic locking — read current version of the aggregate, update if version unchanged, else retry.

**Interviewer:** Pointed out (a) DynamoDB `UpdateItem ADD` is an atomic server-side increment — no lock/CAS needed; (b) the real problem: a viral ad = one adId = one partition = one record taking ~2M increments/s, far past DynamoDB's ~1000 writes/s per partition; optimistic locking would retry-storm. How to redesign so the hot key survives?

**Aayush:** Shard the hot adId's data across multiple nodes to distribute write load across multiple counters; adds scatter-gather complexity at read time but within the read latency budget.

**Interviewer:** Correct. Pushed further: even sharded, that's 10M DB writes/s — is there an approach to reduce the number of DB writes by orders of magnitude? What sits between Kafka and the DB?

**Aayush:** Aggregation workers reading Kafka events and aggregating for 1 hour before writing to the DB, greatly reducing write frequency. Must back up the aggregation worker's state to local disk to recover from failures.

**Interviewer:** Credited self-raising state durability. Two follow-ups: (1) a 1-hour window violates the <1min freshness NFR — what window do you flush on? (2) Under at-least-once, if a worker flushes a rollup then crashes before committing its Kafka offset, it reprocesses and double-counts — what makes the aggregation correct?

**Aayush:** (1) 1-minute window — the 1-hour figure was a mistake. (2) Also record the Kafka offset.

**Interviewer:** Noted "record the offset" alone doesn't fix the ordering gap (flush then crash before commit still reprocesses). Asked: given a 1-minute window is a complete recomputation, what could the worker write so a second write is harmless?

**Aayush:** Write the absolute count for the window, so even if the same events are processed again the count stays the same.

**Interviewer:** Confirmed — idempotent absolute-count-per-(key, window) PUT makes at-least-once effectively-once in the stored result; DB rolls per-minute buckets into hour/day. Rendered the deep dive and moved to feedback.

---

## Design Summary

**Requirements Gathered:**
- FRs: ingest click events from publisher sites; dashboard of aggregated metrics per ad/campaign over time ranges by dimension (site, region); near-real-time; retention raw 30d / aggregated 1y.
- NFRs: 99.99% availability for ingestion; <1 min click→dashboard freshness; no click lost (billing-grade durability); query p99 <1s; 1M/s avg, 10M/s peak, 1KB/event (~86 TB/day raw, ~2.6 PB/30d).

**High-Level Architecture:**
- Write: Publisher → Ingestion Service → Kafka (durable, at-least-once, no ordering) → Worker consumer group (idempotent insert via Idempotency-Key) → DynamoDB (PK=adId, SK=timestamp) storing ClickEvent rows + AggregatedMetric rows (hour/day per dimension).
- Read: Dashboard → Query Service → DynamoDB (precomputed AggregatedMetric rows for interval + range).

**Key Design Decisions & Trade-offs:**
- Kafka for durability + buffering + at-least-once; no ordering needed.
- DynamoDB over SQL for write throughput; partition by adId, sort by timestamp.
- Cursor pagination (timestamp+id) for stable reads under inserts.
- (Initially) distributed/optimistic lock for atomic aggregate updates — corrected toward atomic increments.
- Deep dive: write-shard hot keys + scatter-gather; pre-aggregation tier with 1-minute tumbling windows; idempotent absolute-count-per-window writes for exactly-once-effect; checkpoint worker state.

**Scalability & Fault Tolerance Points:**
- Hot-key (viral ad) sharding with read-time scatter-gather.
- Pre-aggregation collapses 10M writes/s into far fewer DB writes.
- 1-minute window to meet freshness SLA.
- Idempotent windowed writes survive at-least-once replay.
- Aggregation-worker state checkpointing.

**Gaps / Missed Areas:**
- Manufactured a locking problem; missed that atomic counters / idempotent windowed writes remove the need.
- Did not self-raise hot key, per-event write volume, or at-least-once double-count — all interviewer-prompted.
- Vague read response shape (no dimension value per row); initially omitted adId on write.
- No hot-key *detection* strategy (vs over-sharding all keys).
- No monitoring/staleness alerting, consumer-lag/backpressure, reconciliation (raw vs rollup), or cost discussion.
- Pace: 65 min, over the 45–50 budget.

---

## Feedback Given

**Requirements Clarification — 4.5/5** — Strong; derived scale numbers independently with arithmetic; set up durability-vs-availability tension. Needed one nudge to commit the list.

**Core Entities — 3.5/5** — First cut too thin (no adId link, no aggregate entity); filled in well once prompted, but should have been obvious from FR2.

**API Design — 4/5** — Idempotency key (unprompted) and cursor pagination with senior justification. Gaps: initially omitted adId on write; vague read response shape (no dimension value per row).

**High-Level Architecture — 4/5** — Clean, well-justified write path (Kafka durability, DynamoDB over SQL, partition/sort keys); read path leans correctly on write-time precompute.

**Component Design & Trade-offs — 3.5/5** — DynamoDB-vs-SQL justified. Distributed-lock decision a genuine misstep: reached for a lock to do +1, escalated to optimistic locking when atomic counters make it unnecessary.

**Scalability & Fault Tolerance — 4/5** — Correct hot-key sharding once pushed; self-raised state durability. Didn't drive to the breaking case himself.

**Deep Dive Quality — 4/5** — Good arc ending in the right place (sharding → pre-aggregation → 1-min windows → idempotent absolute-count writes), but the "break" was interviewer-supplied each time; 1-hour-window slip contradicted his own NFR.

**Communication — 4/5** — Clear and well structured; main issue is pace.

### Senior Readiness Debrief

**Scorecard:**
- Owns the narrative / self-raises traps: **Weak** — hot key, write volume, at-least-once trap all interviewer-prompted; self-raised only state durability.
- Leads with trade-offs vs named alternatives: **Mixed** — DynamoDB-vs-SQL and scatter-gather good; lock decision had no alternative considered.
- Pushes scale until it breaks: **Mixed** — capable once pushed, didn't self-drive past comfortable case.
- API as a designed contract: **Mixed** — idempotency + cursor strong; missing adId, vague response shape.
- Operability / second-order concerns: **Mixed** — self-raised checkpointing; missed hot-key detection, monitoring/staleness, cost.
- Pace: **Weak** — 65 min vs 45–50; lock detour and slow entity/HLD ramp ate the budget.

**Overall:** Solid **hire**, mid-to-senior boundary — not strong-hire this round. Design ended correct and deep but was interviewer-led to the hard insights; lock detour was a real error. Regression vs prior rounds' mostly-Strong signals; watch it.

**What a senior strong-hire does on this problem:** self-raises the per-event-write problem and goes straight to stream pre-aggregation; never introduces a lock for a counter (atomic increment / idempotent absolute-count windows); self-raises the hot key with detection + write-sharding; self-raises exactly-once (absolute count per window, not increment); covers operability — consumer lag/backpressure, freshness SLO alerting, raw-vs-rollup reconciliation, fleet cost.

**Drill:** `system_design_senior_guidance.md`, especially #1 (own the narrative) and #3 (push scale until it breaks). Highest-leverage change: after stating the HLD, attack it yourself before the interviewer does.

**Time Taken: 65 minutes**
