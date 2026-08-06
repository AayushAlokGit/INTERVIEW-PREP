# System Design Round Transcript
**Date:** 2026-08-06
**Start Time:** 14:12:36 · **End Time:** 14:52:51 · **Duration:** 40 min
**Problem:** Distributed Logging & Telemetry Pipeline
**Performance Rating:** 1/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->

## Phase Timings
| Phase | Budget | Actual | Hit? |
|---|---|---|---|
| Requirements | by 14:20 (8 min) | 14:38 (26 min) | No — 3x over |
| Core entities | by 14:24 (12 min) | never delivered | No |
| API design | by 14:29 (17 min) | never delivered | No |
| High-level design | by 14:39 (27 min) | 14:51 (39 min), logs path only | No |
| Deep dive | by 14:52 (40 min) | never reached | No |

Requirements consumed 65% of the round. Round ended when he said "i give up" at the scale-break question.

---

## Conversation Log

**Interviewer [14:12]:** Presented the problem — design a distributed logging & telemetry pipeline ingesting logs, metrics and traces across a large fleet, storing them, and serving debugging / dashboards / alerting. 45-minute budget stated with per-phase deadlines. Gave the draw.io canvas path. Asked for requirements, FRs then NFRs with numbers, plus what's out of scope.

**Aayush:** i have a few questions -> 1. When do we need to raise alerts, when only a particular metric cross a specified threshold? 2. Dashboards will be for the metrics over time?

**Interviewer [14:15]:** Confirmed threshold-over-window alerting (e.g. p99 latency > 500ms for 5 min), downstream paging out of scope, log-pattern alerting his call. Confirmed dashboards are time-series with 1h/24h/7d windows, filters and grouping by service/region/host/endpoint. Named the third read surface: ad-hoc log/trace search during an incident. Asked for FRs and NFRs with numbers.

**Aayush:** can you give an example of what a metric and log and trace woudl look like

**Interviewer [14:19]:** Gave concrete examples of a metric (name, ts, value, tags), a log (ts, level, service, host, trace_id, message, fields) and a trace (trace_id + span tree with parent pointers). Noted trace_id appears on the log as the correlation key. Pushed for FRs/NFRs/out-of-scope.

**Aayush:** FRs -> 1. Ingest logs, metrics, traces from service machines across a company's fleet. 2. Store the logs, metrics and traces durably. 3. Allow engineer to query the metrics timeseries and logs and traces. 4. Support dashboards on metrics (last 1h / 24h / 7d). 5. Support raising alerts on metrics

**Interviewer [14:19]:** FRs clean. Asked for out-of-scope and the NFR numbers — fleet size, events/sec by type, event sizes, bytes/day, retention, storage totals, query latency, freshness. Told him to pick the numbers himself and show the arithmetic.

**Aayush:** can you help me with some scale estimates, here are the remaining NFRs -> 1. Highly available for event ingestion (99.99 ~ 52 mins downtime/year) 2. Data freshness latency ~ 3 min (an event reflected in dashboard within 3 min) 3. Alert raised within 1 min of metric threshold being breached. 4. Low latency query time < 1min.

**Interviewer [14:25]:** Declined. Told him to pick his own numbers — hosts, services, series per host and scrape interval, log lines/sec and line size, trace sampling rate — state them as assumptions and multiply through to events/sec and bytes/day, then retention to total storage. Noted requirements were 13 min in against an 8-min budget.

**Aayush:** need help with the scale numbers will derive calculation from there

**Interviewer [14:28]:** Supplied the raw inputs (10,000 hosts, 500 services, 1M req/s peak; metrics 200 series/host at 10s and ~100 B/point; logs 100 lines/s/host at ~500 B/line; traces 1% sampled, ~10 spans, ~300 B/span) and made him choose retention per pillar and justify the difference. Demanded events/sec and bytes/day per pillar, total ingest, and total storage. Stated the phase would be cut as soon as the numbers arrived.

**Aayush:** Metrics -> 2*10^5 events/s ~ 20 GB/s. Logs -> 10^6 logs/s ~ 500GB/s. why is there need for sampling in traces ?

**Interviewer [14:34]:** Answered the sampling question (10M spans/s unsampled = 3 GB/s, several times the log volume, mostly identical successful requests; the interesting question is *which* 1% you keep). Then challenged the byte figures: "You said logs are 500 GB/s. That's 43 petabytes a day. Re-run that multiplication and tell me if you still believe it."

**Aayush:** yes logs figure checks out. traces 300GB/s. retention per pillar is 15 dys for logs and traces

**Interviewer [14:38]:** Corrected the arithmetic explicitly — 10^6 x 500 B = 500 MB/s not 500 GB/s (1000x), metrics 20 MB/s not 20 GB/s (1000x), traces 30 MB/s not 300 GB/s (10,000x). Gave the corrected table: 1.3 M events/s, ~550 MB/s, ~47 TB/day; 650 TB logs and 39 TB traces at 15-day retention. Noted that at his figure nothing he designed would have been real. Asked for metrics retention and why it should differ. Closed requirements and demanded entities + API + HLD.

**Aayush:** metrics can be retained for shorter times, because the metrics aggregates can be stored for past windows

**Interviewer [14:39]:** Credited rollups/downsampling as the right mechanism but flagged that he concluded metrics should be retained *shorter* when metrics are the cheapest pillar (1.7 vs 43 TB/day). Asked for a raw-metrics and rolled-up-metrics retention number. Re-demanded entities (including how a metric series is identified), the ingest contract, the three read contracts with shapes and pagination, and the HLD.

**Aayush:** update drawio file

**Interviewer [14:40]:** Rendered FRs, NFRs and the scale table to the canvas, marking out-of-scope as unstated. Noted 28 minutes in with no design and asked for entities + API + HLD in one pass.

**Aayush:** i am blanking out

**Interviewer [14:43]:** Narrowed the frame — take one single log line written on host vm-1a3f, an engineer needs to find it by trace_id in two minutes; walk it one hop at a time until it's queryable. Told him to ignore metrics, traces and alerting for now.

**Aayush:** the log needs to be pushed to our ingesting service which will write the logs to a datastore which will should be optimised for quering huge log volumes

**Interviewer [14:45]:** Called it a spine and asked three questions: (1) what on the host pushes it, and push or pull? (2) name the datastore and what it's being picked over — "optimised for querying huge log volumes" is a property, not a choice. (3) the ingest service is absorbing 550 MB/s and the datastore has a bad ten minutes — what happens to that log line?

**Aayush:** there must be a program running on client devices which will aggregate the logs, and keep track of which logs have been sent. It will push the logs to the ingesting service. The logs would be pushed to a kafka queue paritioned by machineId so ordering of logs of a machine is retained. There would be consumer process who would poll kafka and write the logs to a OLAP DB.

**Interviewer [14:51]:** 39 min in, 6 left. Two final questions: (1) name the OLAP DB and its alternative. (2) The scale push — a bad deploy at 14:52 takes one service's 500 hosts from 100 to 10,000 log lines/sec, ~250 MB/s from that service alone, exactly when engineers need the logs most. Given Kafka partitioned by machineId and consumers writing to the OLAP DB: what breaks first, and what does the engineer querying at 14:54 actually see?

**Aayush:** i give up

**Interviewer [14:52]:** Round called at 40 minutes.

---

## Design Summary

**Requirements Gathered:**
- 5 FRs, clean and correctly scoped, delivered at 14:19. Best part of the round.
- 4 NFRs with numbers attached: 99.99% ingest availability, 3-min freshness, 1-min alert, <1-min query. Better than qualitative NFRs, but never reconciled — NFR 2 (3 min freshness) and NFR 3 (1 min alert) are in direct contradiction if the evaluator reads the same store the dashboards read. He never noticed.
- Zero self-derived scale numbers. Asked for help twice; interviewer supplied the raw inputs.
- Arithmetic wrong by 1000x on metrics and logs, 10,000x on traces; defended the log figure when challenged directly.
- Retention: 15 days logs and traces. Metrics retention never given; reasoned they should be retained *shorter*, which is backwards (cheapest pillar).
- Out of scope: never stated.

**High-Level Architecture (logs path only):**
`Agent on host (aggregates, tracks what has been sent)` → `Ingesting Service` → `Kafka, partitioned by machineId for per-machine ordering` → `Consumer process (polls)` → `OLAP DB (unnamed)` → `Engineer query`

Metrics, traces and alerting never got a path. Dashboard read path never drawn. The alerting FR — one of his own five — was never designed.

**Key Design Decisions & Trade-offs:**
- Push from a host agent rather than pull. Correct, but no alternative named.
- Kafka as a buffer between ingest and store. Correct instinct, the strongest moment of the round.
- Partition by machineId to preserve per-machine log ordering. Reasoned, but the wrong key for this workload — it doesn't spread a single service's storm, and consumers scale by partition count.
- "OLAP DB" — a category, never a named system, never an alternative.
- No named alternative for any component all round.

**Scalability & Fault Tolerance Points:**
None demonstrated. Asked directly twice — what happens when the datastore stalls for ten minutes, and what breaks in the 100x log storm — and answered neither.

**Gaps / Missed Areas:**
- Core entities: never produced (asked three times)
- API design: never produced (asked three times)
- Metrics ingest/query path, trace path, alerting path: never designed
- MetricSeries vs MetricPoint separation — the highest-leverage entity decision in a telemetry system
- Metric cardinality explosion
- Idempotency under at-least-once agent retry
- Consumer lag / backpressure / load shedding
- Hot partition analysis on his own partition key
- Freshness reporting (`max_ts_ingested`) — the system silently lying when behind
- Tail vs head sampling for traces
- Cost at target scale
- Monitoring the monitoring system

---

## Feedback Given

### Phase timings
Requirements consumed 65% of the round. That single fact determines most of what follows.

### What went wrong

**He asked the interviewer to do the sizing, twice.** "Can you help me with some scale estimates" at 14:25, then "need help with the scale numbers" after pushback. The inputs were supplied anyway so the round could continue, but in a real interview that exchange *is* the round — watching the candidate invent and defend assumptions is the evaluation. He had everything he needed at 14:19.

**The arithmetic was wrong by 1000x and 10,000x, and he defended it.** Asked directly to re-run the multiplication, he replied "yes logs figure checks out," then produced a traces figure off by another four orders of magnitude. 500 GB/s is 43 PB/day from 10,000 hosts. A senior candidate feels that is wrong before checking, because they carry reference points: a NIC is 10-25 Gbps, a fat Kafka broker takes ~1 GB/s, a big S3 bucket is petabytes total. The check isn't the multiplication, it's the smell.

**No entities, no API, ever.** Asked three times including an explicit list at 14:39.

**The HLD arrived at 14:45 and only for logs.** The skeleton is correct and the Kafka partition-key reasoning was a real instinct. But metrics, traces and alerting — one of his own FRs — never got a path, and the dashboard read path, the highest-volume read surface, was never drawn.

**He gave up on the scale question**, which was the senior-differentiating question of the round and was aimed squarely at his own design.

### Rubric

| Dimension | Assessment |
|---|---|
| Requirements — FRs | Good. Five clean FRs, correctly scoped, fast. Best part of the round. |
| Requirements — NFRs | Weak. Four NFRs with numbers — better than "highly available" — but zero self-derived scale, failed arithmetic, no out-of-scope, unreconciled contradiction. |
| Core entities | Absent. |
| API design | Absent. |
| High-level architecture | Weak-to-fair. Correct spine for one of three pillars. |
| Component design & trade-offs | Weak. "OLAP DB" never named. No alternative named for any component. |
| Scalability & fault tolerance | Not demonstrated. Two direct questions, no answers. |
| Deep dive | Never reached. |
| Communication | Weak. Two explicit requests for help, one "blanking out," one "I give up." |
| Diagram quality | Requirements rendered; no architecture beyond a four-box chain. |

### Senior-signal scorecard

| Signal | Status | Why |
|---|---|---|
| Owns the narrative / self-raises traps | Weak | Every trap was the interviewer's. Zero self-raised. |
| Leads with trade-offs vs named alternatives | Weak | Not one named alternative all round. |
| Pushes scale until it breaks | Weak | Wouldn't derive the comfortable case, gave up on the break case. |
| API as a designed contract | Weak | No API produced. |
| Operability / second-order concerns | Weak | Backpressure unanswered; no monitoring, cost, or hot-partition analysis. |
| Pace | Weak | Requirements at 3x budget; deep dive never reached. |

**Overall read: mid-level, no-hire on this round.** Not a read on ability — the DSA rounds the same day showed real modeling instinct, and the Kafka-partitioned-by-machineId answer came from someone who has genuinely worked on pipelines. A read on this performance.

**Performance Rating: 1/5** — the architecture had to be extracted question by question, and three of five FRs never got a design.

### What a senior strong-hire would have done on THIS problem

**Self-raised the NFR contradiction.** "Freshness ~3 min" and "alert within 1 min" one line apart cannot both hold if the alert evaluator reads the store the dashboards read. The senior move is catching that yourself and resolving it: the evaluator consumes the Kafka metrics topic directly, bypassing storage. That single observation carries the round.

**Separated the pillars and justified it.** Three topics — logs by `hash(service, host)`, metrics by `hash(series_id)`, traces by `hash(trace_id)` — because they differ in volume by 25x, in retention, and in consumer, and because a log storm must never delay the metrics that carry the alerting SLA.

**Split MetricSeries from MetricPoint.** Tags live on the series; a point is `(series_id, ts, value)`. That turns 100 B on the wire into ~2 B at rest and makes 13-month metric retention affordable — and it's why the instinct to retain metrics *shorter* was backwards.

**Named the store and its alternative.** ClickHouse over Elasticsearch for logs (at 43 TB/day an inverted index costs about as much as the data). A purpose-built TSDB over ClickHouse for metrics (Gorilla encoding is a 50x win a general columnar store can't replicate).

**Answered the storm question.** Gateway quota with `429 + retry_after_ms`, shed DEBUG before ERROR, autoscale consumers on lag not CPU, over-provision partitions up front — and above all return `max_ts_ingested` on every query so the UI says "data delayed 4 min" instead of showing an empty chart that reads as "everything is fine." That is the difference between a tool on-call trusts and one they don't.

Pointed him to the pre-round self-check in `system_design_senior_guidance.md`.

### Optimal reference design
Appended to `logging_and_telemetry_pipeline.drawio` as Part B, below his own design (Part A, rendered faithfully with gaps marked): core entities, full API contracts for ingest / metric query / log search / alert-rule CRUD, complete HLD across edge → ingest → buffer → processing → storage → serving, key trade-offs each against a named alternative, and deep dives on the 100x log storm, cardinality explosion, idempotency, late events, clock skew, monitoring the monitoring, and cost.
