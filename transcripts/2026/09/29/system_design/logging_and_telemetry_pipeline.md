# System Design Round Transcript
**Date:** 2026-09-29
**Start Time:** 17:58:56 · **End Time:** 19:02:30 · **Duration:** 64 min
**Problem:** Logging & Telemetry Pipeline (logs only; alerting over log content)
**Difficulty:** 3/5 (Medium). One real scale break: full-text search over a 1.5 PB, 30-day hot window under a 5 s p99, where the naive time-partitioned store must become an indexed or tiered one. One genuine trade-off: index everything (Elasticsearch) vs index labels and scan chunks (Loki / ClickHouse), which is cost vs query latency. Plus alert evaluation that must not hammer the search store.
**Dominant pattern:** scaling writes (1M lines/s ingest), with a scaling-reads break on search and a realtime-updates element on alerting
**Performance Rating:** 2/5  <!-- machine-read on future rounds; ≤2 = eligible for re-ask, ≥3 retired -->

**Would it have fit a real 45-min round?** No. Cut off in the high-level design (HLD only finished at +42m); the deep dive would never have started.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 15 min (18:14:17) | +7 | Over |
| Core entities | 12 min | 18 min (18:16:57) | +6 | Over (entities themselves took ~3 min) |
| API design | 17 min | 26 min (18:24:53) | +9 | Over |
| High-level design | 27 min | 42 min (18:40:28) | +15 | Over |
| Deep dive | 40 min | 62 min (19:00:38) | +22 | Over |
| Wrap-up | 45 min | 64 min (19:02:30) | +19 | Over |
| **Total** | 45 min | 64 min | +19 | Over |

## Round-trip Tax
| Phase | Parts asked | Answered 1st pass | Follow-up exchanges | Minutes lost | What was missing |
|---|---|---|---|---|---|
| Requirements | FRs, NFRs with numbers | FRs; NFRs partially | 2 | ~4 | Storage sizing; retention totals (never given); read arithmetic (wrong twice, steps never shown) |
| Core entities | Entities | Yes | 0 | 0 | Fired-alert state; service field on Log |
| API design | Endpoints + shapes | 3 of 3 endpoints | 2 | ~4 | Alert evaluation window (never given); PATCH path; error/backpressure semantics |
| High-level design | Ingest, query, alerting paths | Ingest and alerting only | 1 | ~4 | Query path; what alert workers query; storage trade-off |
| Deep dive | Query break → fix → scale → alerting | Break identified | 6 | ~20 | Index mechanism and size; ES vs ClickHouse trade-off (asked twice, never answered); cluster sizing; 10× fix; alert redesign ("don't know") |
| **Total** | | | 11 | ~32 of 64 min | |

**Deferrals used:** 0.

---

## Conversation Log
**Interviewer (17:58:56, +0m):** Design a logging and telemetry pipeline. Thousands of services emit logs and telemetry; engineers need to collect centrally, search when debugging, and be alerted when something goes wrong. Difficulty 3/5 (Medium). Stated the reference timeline (measured, not enforced) and the canvas path. Asked him to start with requirements.
**Aayush:** Are metrics also part of the system, or does it just deal with logs from machines?
**Interviewer (18:00:30, +2m):** Logs only. Metrics/time-series is out of scope. Alerting in scope means alerts over log content (e.g. more than N ERROR lines from service X in 5 minutes).
**Aayush:** FRs: 1. Collect logs from a fleet of machine hosts. 2. Store collected logs. 3. Query logs within a time range, and by content string matching. 4. Alerts based on logs.
**Interviewer (18:02:39, +4m):** FRs are on the canvas. Now the NFRs.
**Aayush:** What are the scale numbers?
**Interviewer (18:04:12, +5m):** 100,000 hosts · 1M lines/host/day avg · 500 B/line · peak 3× · 30 days searchable, 1 year archive · 5,000 engineers × ~100 searches/day · 10,000 alert rules.
**Aayush:** NFRs: 1. Highly available ingestion; lower availability for querying acceptable. 2. Query p99 < 5 s. 3. Durably store up to 1 year; last 30 days queryable, older to cold storage. 4. 100k hosts × 1M lines/day = 10 lines/host/s → 1M lines/s avg, 3M peak. Reads: 500 engg × 100 searches/day × 100 lines each → 50M lines/s. Read heavy. 5. Alerts within 2 min of the trigger event.
**Interviewer (18:10:29, +12m):** Walk through the read arithmetic step by step. Also size storage per day and across retention.
**Aayush:** 1. 5M log lines/s, so read:write 5:1. 2. Ingest 1500 MB/s peak, 150 TB/day.
**Interviewer (18:12:38, +14m):** Numbers, not steps. Show engineers → searches/day → searches/s → lines/s. For storage: which rate gave 150 TB/day, and what are the 30-day and 1-year totals? (Use a day ≈ 10^5 s.)
**Aayush:** Read steps are in the previous message. Storage: 3M lines/s peak × 500 B × 10^5 s ≈ 150 TB/day.
**Interviewer (18:14:17, +15m):** NFRs are on the canvas as stated. Next: core entities.
**Aayush:** 1. Log (machineHostId, logMsg, createdAt). 2. AlertRule (conditionsToFireAlert, notificationSubscribers).
**Interviewer (18:16:57, +18m):** On canvas. Next: API design.
**Aayush:** POST /logs, Idempotency-Key header, batch body {machineId, logs:[{createdAt, msg}]}, 2xx = accepted. POST /alerts, Idempotency-Key, {notificationSubscribers:[], alertRules:{}}. GET /logs?from&to&query&cursor&limit → {Log[], nextCursor}; cursor pagination because log lines are inserted dynamically.
**Interviewer (18:21:01, +22m):** What fields does a rule contain? Not blocking: what does POST /alerts return, and how is a rule changed or deleted later?
**Aayush:** logQuery + threshold + comparison operator. POST returns the alert object; a PATCH endpoint updates a rule.
**Interviewer (18:24:03, +25m):** The threshold is compared against what value, computed from logQuery how?
**Aayush:** logQuery must produce a number to compare with the threshold; only then can the system decide whether to fire.
**Interviewer (18:24:53, +26m):** API on canvas. Next: HLD, covering ingest, query and alerting.
**Aayush:** An agent on each host batches logs and POSTs them periodically. The API server puts the message on Kafka for durability and returns success immediately. Kafka is partitioned by machineHostId for per-machine ordering. Consumers write to a high-write-throughput DB with querying, like Cassandra, partitioned by createdAt. That creates a hot shard on heavy days, so write load is spread dynamically across shards, at a cost in read time. Alerting: background workers repeatedly run each rule's logQuery, every 2 min, and notify subscribers if it fires.
**Interviewer (18:36:38, +38m):** Drawn. Three gaps: which component serves GET /logs and from what; where AlertRules are stored and what workers run logQuery against; Cassandra vs which alternative, and what is given up.
**Aayush:** 1. Another read API queries the DB. 2. AlertRules in the same Cassandra; no special querying needed. 3. SQL can't handle the workload. With Cassandra we give up richer querying, but only createdAt ordering is needed, handled by the sort key.
**Interviewer (18:40:28, +42m):** Added. Alert workers still unconnected. Deep dive: a 7-day query for "timeout connecting to payments-db". What does Cassandra do, and does it meet 5 s p99?
**Aayush:** It touches partitions for the past 7 days, about 150 TB × 7, so it won't meet the SLA.
**Interviewer (18:42:38, +44m):** Right. What do you change?
**Aayush:** We need proper text indices. Cassandra lacks them; ClickHouse has them and handles high write throughput, so it's the better choice.
**Interviewer (18:51:25, +52m):** 1. Mechanism: what does the index store, how does the query use it, how much is read now? 2. Why ClickHouse over Elasticsearch/OpenSearch, and what do you give up?
**Aayush:** 1. Inverted text indices mapping keywords → log lines. 2. An Elasticsearch index can hold up to 30 days, kept updated by the Kafka consumers. Logs also go to S3 for durable storage; ES gives the query flexibility.
**Interviewer (18:54:20, +55m):** Does ES replace ClickHouse and Cassandra? And what does ES cost you vs ClickHouse?
**Aayush:** ES replaces ClickHouse and Cassandra as the log store serving GET, kept updated by the Kafka consumers, which also write to S3. [follow-up] Another DB for storing alertRules; simple PostgreSQL will work.
**Interviewer (18:57:01, +58m):** Deep-dive section drawn. Scale: how much data sits in ES for 30 days, and how many nodes/shards? At 10× volume, what breaks first and what do you do? Alerting: 10,000 rules every 2 min. What query rate, and does it survive?
**Aayush:** 1. 150 TB × 30, which is huge, so many shards. At 10×, query times break first. 2. 5,000 log queries/s; at 100 lines each, 500k lines/s from alert load.
**Interviewer (18:59:43, +61m):** Both stop at the break. At 10×, what do you change? Alerting at your 5,000 q/s on the same ES serving engineers: does it survive, and how do you redesign it?
**Aayush:** Don't know.
**Interviewer (19:00:38, +62m):** Wrap-up: main bottlenecks and failure points, and how you'd know the pipeline is falling behind or losing logs.
**Aayush:** Queue depth shows how far behind the pipeline is; scale workers on queue depth.
**Interviewer (19:02:30, +64m):** Round ended; feedback given.

---

## Design Summary
**Requirements Gathered:**
- FRs: collect, store, query by time range + content, alerts.
- NFRs: HA ingest, 5 s p99 query, 1-year durability with 30 days searchable, 2-min alert SLA.
- Scale: 1M lines/s avg, 3M peak (correct).
- Stated but wrong: reads "50M, then 5M lines/s, read-heavy 5:1"; storage "150 TB/day" (used peak).

**High-Level Architecture:**
- Ingest: host agent (batching) → API server (ack after enqueue) → Kafka (partitioned by host) → consumers → Cassandra by createdAt.
- Deep dive replaced Cassandra with Elasticsearch (30 days) + S3 (cold/durable) + PostgreSQL (AlertRules).
- Query: Read API → store.
- Alerting: workers poll each rule's logQuery every 2 min → notify subscribers. The data source was never stated.

**Key Design Decisions & Trade-offs:**
- Kafka for durable, async ingest with an immediate ack.
- Batching at the agent.
- Idempotency key on ingest.
- Cursor pagination.
- Cassandra over SQL for write throughput, later abandoned.
- Elasticsearch for full-text search, with no trade-off stated.

**Scalability & Fault Tolerance Points:**
- Self-raised the createdAt hot shard.
- Consumer lag as a health signal (prompted, at wrap-up).

**Gaps / Missed Areas:**
- Read load wrong by 10^4 and read:write direction wrong: the system is overwhelmingly write-heavy.
- Daily storage 3× high, and retention totals never computed.
- No ES sizing, and no 10× plan.
- Alert evaluation: rate wrong (83/s actual vs 5,000 stated); no window in the rule; runs against the search store; no fired-state/dedupe; no redesign.
- No agent-side buffering or backpressure.
- No 429/overload semantics.
- No DLQ.
- No noisy-host / hot Kafka partition handling.
- No loss detection or freshness monitoring.
- No cost consideration at PB scale.
- No out-of-scope statement.

---

## Feedback Given

### Pace report
| Phase | Ref | Actual | Delta |
|---|---|---|---|
| Requirements | 8 | 15 | over by 7 |
| Core entities | 12 | 18 | over by 6 |
| API design | 17 | 26 | over by 9 |
| HLD | 27 | 42 | over by 15 |
| Deep dive | 40 | 62 | over by 22 |
| Wrap-up | 45 | 64 | over by 19 |

**Would it have fit a real 45-minute round? No.**
- Your HLD was only complete at +42m. A real interviewer would have had three minutes left and would never have reached the deep dive, which is where this problem is decided.
- **The overrun was your process, not the problem's size.** This is a 3/5 problem.
- **Biggest time sink: the HLD phase (16 minutes).** It went 12 minutes quiet, then came in missing the query path. The deep dive was the next biggest: 20 minutes over six exchanges.
- **Round-trip tax:** 11 follow-up exchanges, about 32 of 64 minutes. It's extraction, not thinking. Most follow-ups asked for something you'd been asked for in the first prompt, such as the query path, the storage totals and the arithmetic steps.
- **Deferrals:** 0.

### Senior-signal scorecard
| Signal | Read | Why |
|---|---|---|
| Owns the narrative | **Mixed** | You self-raised an idempotency key on batch ingest, cursor pagination with a reason, the ack-after-enqueue durability point and the createdAt hot shard. That's genuinely good. But the HLD shipped without a query path, and the alert workers never had a data source. |
| Leads with trade-offs | **Weak** | Cassandra vs SQL was a real comparison, but you justified it by saying "only createdAt ordering is needed", which contradicts your own FR3 (content matching). ClickHouse vs Elasticsearch was asked twice and never answered. |
| Pushes scale until it breaks | **Weak** | Three arithmetic errors, all uncaught. You correctly spotted the Cassandra scan break, but stopped at "many shards" for sizing and said "don't know" at 10×. |
| API as a designed contract | **Mixed** | Batch ingest, Idempotency-Key and `nextCursor` with a reason are strong. The alert rule has no evaluation window, the PATCH has no path, and there are no overload semantics (`429` / retry-after) on an endpoint taking 3M lines/s. |
| Operability | **Weak** | Consumer lag came up only at wrap-up, when prompted. You didn't cover how to detect lost logs, a noisy-host hot partition, agent-side buffering when ingest is down, a DLQ, alert dedupe or cost. |
| Pace | **Weak** | The core design finished at 42m and the round took 64m. |

**Overall:** mid-level, no-hire at senior. It's a clear step up from the last sitting of this problem, which was a 1/5 and never got past the core design.

### Performance Rating: 2/5
You reached a coherent design and found the central break yourself. But the numbers that set the problem were wrong in the direction that matters: this is a write-heavy system, not a read-heavy one. The two scale questions ended in "don't know", and alerting has no workable mechanism.

### What a senior strong-hire would have done on THIS problem
**1. Numbers, with a day = 10^5 s.**
- Writes: 10^6 lines/s average, 3×10^6 at peak.
- Bandwidth: 500 MB/s average, 1.5 GB/s peak.
- Storage: **50 TB/day** raw. The 30-day window is **1.5 PB** raw; with replicas and index overhead, Elasticsearch would hold about 3 PB. One year is about 18 PB raw, or about 2 PB compressed in S3.
- Reads: 5,000 × 100 = 5×10^5 searches/day, which is **about 5 searches/s**.
- Conclusion: **massively write-heavy by count, but each read is expensive.** That sentence should decide the architecture: optimise the ingest path for throughput and cost, and make each search scan as little as possible.

**2. Scoping.** Name the out-of-scope items up front: metrics, tracing, PII scrubbing, multi-tenant billing.

**3. Self-raised traps.**
- **Agent-side buffering.** The agent writes to local disk and retries with backoff when ingest returns `429`. Otherwise "highly available ingest" means dropped logs.
- **Hot partition.** Partitioning Kafka by `machineHostId` puts a noisy host's whole flood on one partition. Per-host ordering isn't needed, because queries sort by timestamp. Hash-partition for balance instead.
- **Idempotency on replay.** The batch key must survive a consumer replay into the index, not just the HTTP retry. Deduplicate on (host, batch id, offset).

**4. The storage trade-off, the actual deep dive.**

| | Index everything (Elasticsearch/OpenSearch) | Index labels only (Loki, or ClickHouse with bloom-filter skip indexes) |
|---|---|---|
| How it works | An inverted index over every token | Index service, host, level and time; store compressed chunks in S3; a content search runs a parallel grep over only the chunks that match the labels and time range |
| Query speed | Fast arbitrary search | Slower on unscoped content search |
| Cost at 1.5 PB/month | The index is about as big as the data, indexing CPU dominates, and cost explodes | Order-of-magnitude cheaper storage, trivial ingest |
| At 10× (about 15 PB hot) | Indexing throughput and cost break first | Holds up, if queries are scoped |

The senior answer at 10× is label-only indexing with a **mandatory service filter plus a time range** in the query API. Add time-based indices or partitions with a lifecycle policy (hot, then warm, then S3), plus per-service ingest quotas.

**5. Alerting.** Never poll the search store.
- 10,000 rules / 120 s ≈ **83 queries/s**. That's 17 times the human search load, and every query is a repeated scan.
- Instead, evaluate rules **on the stream**. A consumer group (or Flink) reads Kafka, routes each line only to the rules for its service, and keeps windowed counters per rule.
- That fires in seconds, well inside the 2-minute SLA, and puts zero load on the search store.
- Fired alerts get a state machine (pending → firing → resolved) for dedupe, plus idempotent notification.
- The rule contract therefore needs `window` and `groupBy`.

**6. Operability.**
- **Freshness:** consumer lag per partition, plus an end-to-end canary. Inject a synthetic log line and alert if it isn't searchable within N seconds.
- **Loss:** drop counters at the agent vs lines received at ingest vs lines indexed, reconciled per host.
- **Failures:** a DLQ for malformed batches, and cluster health checks.
- **Cost:** the $/TB-month per tier.

### Checklist
Go through the quick pre-round self-check in `system_design_senior_guidance.md` before your next round. The two items that cost you most today:
- *"Did I state avg and peak numbers, with the arithmetic, unprompted?"*
- *"Did I push scale until something broke, and design for the broken case?"*

---

## Drill Prescription
| # | Gap targeted (signal / weakness) | Exercise (on which system) | Timebox | Pass criterion |
|---|---|---|---|---|
| 1 | Pushes scale / arithmetic slips: reads 10^4× high and called "read heavy", storage from the peak rate, alert load 60× high | *Numbers:* redo only the NFRs and back-of-envelope for this logging pipeline from the same givens. Write every step with day = 10^5 s: writes/s, bytes/s, TB/day, 30-day and 1-year totals, searches/s, alert evals/s. End on the one sentence that decides the architecture. | 8 min | All derived numbers within 2× of the reference (50 TB/day, 1.5 PB hot, ~5 searches/s, ~83 evals/s); "write-heavy by count, expensive reads" stated; no step skipped |
| 2 | Trade-offs vs named alternatives: ClickHouse vs Elasticsearch asked twice, never answered | *Trade-offs:* a table comparing Elasticsearch, ClickHouse (skip indexes) and Loki-style label-index + S3 chunks for the 30-day search tier: what each gives up, cost at 1.5 PB, which breaks first at 10×, and your pick | 15 min | Three named options, each with what it gives up; a 10× verdict; a pick justified from your own numbers, unprompted |
| 3 | Moves on from a break he identified: "don't know" on alerting at scale | *Scale break:* redesign alert evaluation for 10,000 rules without querying the search store. Compute the eval rate, give the mechanism in 3 bullets, and give the AlertRule contract fields. | 12 min | Correct eval rate; stream-side windowed evaluation; the rule contract has window + groupBy; fired-state dedupe named |
