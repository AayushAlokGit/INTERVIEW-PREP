# Design Sprint Transcript (front half, timeboxed)
**Date:** 2026-08-14
**Start Time:** 11:28:28 · **End Time:** 11:47:06
**Problem:** Logging & Telemetry Pipeline
**Difficulty:** 4/5 — extreme write skew, two storage regimes (logs vs metrics) with different retention, and a given that doesn't survive a plausibility check
**Front-half readiness: 2/5**
**Front half complete inside 17:00: no** — API landed at 18:38; at the buzzer there were no request bodies, no response fields, no error semantics, and no search parameter on the log-search endpoint. Read:write ratio, storage growth, and durability were never stated.

> Sitting stopped after problem 1 of 2 by request.

| Phase | Pace target | Landed at | ± vs target | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 8:00 | 10:50 | +2:50 | 1 (after 2 clarifying round trips) | 2/5 |
| Core entities | 12:00 | 15:57 | +3:57 | 1 | 1/5 |
| API design | 17:00 | 18:38 | +1:38 | 1 | 2/5 |

**Budget allocation:** 4:10 on two serial clarifying round trips, then 6:40 composing requirements — **10:50 to first content**, 64% of the box gone before a single FR was written. Entities took **5:07 to produce three lines / 12 words**, the single largest sink and the weakest phase. API got 2:41 and was the phase where the remaining budget clearly ran out.

**First-pass completeness:** good, and better than the last sitting. Every phase closed in **one** message; nothing was back-filled, nothing needed a second round trip. The failure is not packaging this time — it is throughput on the thinking phases.

**Plausibility check:** **not performed.** He asserted 100 MB/s of log ingest and moved on. The correct derivation is 10,000 services × 100 hosts = 10^6 hosts × 10^3 events/s = **10^9 events/s = 1 TB/s** — he dropped the 10,000-service multiplier, a 10,000× error. One line of division would have caught it from either direction: 100 MB/s across 10,000 services is 10 KB/s per service, i.e. one 1KB log line every ten seconds per service, which is obviously wrong; or 1 TB/s is ~86 PB/day, larger than any real logging system on earth, which is the pushback a senior gives to the given itself.

---

## Problem Statement

> Design a logging and telemetry pipeline. Services across the fleet emit structured log events and metrics; engineers need to search recent logs and chart metrics over time.
>
> **Difficulty: 4/5**

**Clarifications asked:** is alerting part of the system (no) · what time windows must be queryable and what is the retention (logs searchable 7d / retained 30d; metrics queryable to 1y / retained 1y, downsampled after 30d).

**Givens supplied:** 10,000 services · 100 hosts each emitting 1,000 log events/sec/host at 1KB per event · 10M distinct metric series sampled every 10s · peak 3× average · 500 engineers at ~50 queries/day each.

---

## What he produced (verbatim, as it stood at the buzzer)

### Requirements

**FRs**
1. Ingest structured logs and metrics from machine fleets.
2. Allow engineer to query recent logs (last 7 days) and chart metrics (upto last 1 year)

**Out of scope**
1. Alerting
2. Tracing

**NFRs**
1. Highly available for ingestion (99.99 ~52 mins downtime/year)
2. Eventual consistency for the event ingestion.
3. Freshness latency for events (<1s)
4. Latency of query (p99 < 1s)
5. Logs stored for 7 days and metrics for upto 1 year.
6. Scale of 10k services and 100 hosts emitting 1000 log events/sec/host with 1Kb per event -> 100 MB/s for log events. 10M metric series samples every 10s -> 1M metric points/s avg and peak 3x.

### Core entities

1. Log (text, timestamp, hostId)
2. MetricSeries (dimensions)
3. MetricPoints (metricSeriesId, timestamp)

### API design

1. `POST /logs`
2. `POST /metricPoints`
3. `GET /logs?startTime&endTime&hostId`
4. `GET /metricPoints?metricSeriesId&startTime&endTime`

Post APIs will be idempotent using idempotency keys, GET APIs will have cursor based pagination.

---

## What was still missing at 17:00

**Requirements**
- **Read:write ratio.** Both numbers were handed to him (500 engineers × 50 queries/day, and the emitter rates). 2.5×10^4 queries/day ≈ 0.25 QPS against ~10^9 writes/s is a ratio near **1 : 4×10^7** — the most consequential number in this system and the one that dictates the entire storage strategy.
- **Storage growth.** Never computed. At the real rate: ~1 PB/day raw, ~3 PB for 30 days after compression.
- **Durability posture.** Never stated. For telemetry the interesting answer is that logs are *lossy-tolerant* and the system sheds load at the edge rather than blocking emitters — a genuine design decision, skipped.
- **Availability for the query path** — 99.99% was stated for ingestion only.
- **Consistency per subsystem** — one blanket "eventual consistency for ingestion"; nothing for the metric or query path.
- Freshness `<1s` carries no percentile (the query latency correctly does).
- Retention restated as "logs stored for 7 days" when the answer given was **searchable 7 days, retained 30**.

**Core entities**
- **`MetricPoint` has no `value` field** — a metric point that cannot store its metric.
- `Log` has no `serviceId` (10,000 services, no way to filter to one), no severity/level (the most common log filter), and no structured attributes despite FR1 saying "structured logs".
- `MetricSeries` has no name, no unit, no id.
- No keys, no uniqueness constraints, no partition keys on anything.
- **No entity that only appears under load.** Two are forced by his own requirements: a **rollup/downsample** entity (the retention answer explicitly said "downsampled after 30 days", and a 1-year chart is 3×10^6 raw points per series without it) and a **log segment** (at 1 PB/day you index immutable compressed blocks, not individual events).

**API design**
- **No request shapes on either POST.** Zero named fields. Critically: is `POST /logs` one event or a batch? At 10^9 events/s a per-event call is impossible, so the batch size *is* the contract — and it's blank.
- **No response fields anywhere.** Cursor pagination was claimed but `nextCursor` was never named, and cursor-vs-offset was never justified.
- **No error or status semantics.** Nothing on ingest backpressure (429 / retryAfter), nothing on partial batch acceptance.
- **`GET /logs` has no search parameter.** FR2 says engineers *search* logs; the endpoint filters on hostId and time only — no text query, no severity, no serviceId.
- **No series-discovery endpoint.** `GET /metricPoints?metricSeriesId=` requires the caller to already possess a seriesId, and nothing returns one.
- **No `step`/aggregation parameter** on the metric read, so a 1-year query returns millions of points per series. No stated position on payload size for a read that can return a huge collection.
- No DELETE, and no stated reason it isn't needed.

---

## Where the time went

**10:50 of a 17:00 box before the first requirement was written.** Two clarifying round trips (4:10) then composition (6:40). The clarifying questions were good ones — alerting scope and retention windows both changed the design — but they arrived serially, and each cost a full round trip.

**The entities phase is the anomaly: 5:07 for three lines totalling twelve words.** That time did not buy quality — the phase shipped an entity that cannot hold its own value, and omitted both entities his own retention requirement forces. Five minutes of deliberation produced less than a mechanical template would have.

The API phase then got 2:41 and landed 1:38 past the buzzer, which is exactly where the skeleton-without-bodies outcome comes from. The structure was right and volunteered unprompted; there was no budget left to fill it in.

---

## Ideal front half (writable in the same 17 minutes)

### Requirements

**FRs**
1. Ingest structured log events and metric samples from fleet hosts.
2. Search logs over the last 7 days by service, host, severity, time range, and message text.
3. Query and chart metric series over windows up to 1 year, at a caller-chosen resolution.

**Out of scope:** alerting, distributed tracing, log-derived metrics, access control, retention-policy management, cost attribution.

**NFRs**
- **Availability:** ingest 99.99% (52 min/yr) — dropping telemetry during an incident is exactly when you need it. Query 99.9% is fine; engineers tolerate a search outage.
- **Consistency:** ingest is eventually consistent — an event is searchable within 30s of emission. Metrics are approximate under load by design (sampled). No cross-entity consistency requirement anywhere.
- **Latency:** ingest accept p99 < 50ms (fire-and-forget from the agent). Log search p99 < 2s over 7 days. Metric query p99 < 1s over 30 days, < 5s over 1 year.
- **Durability:** logs are **lossy-tolerant** — under overload the edge sheds load rather than blocking emitters, target < 0.01% event loss. Metrics are lossy by nature. Nothing here is a system of record.
- **Traffic model:**
  - Hosts: 10^4 services × 10^2 hosts = **10^6 hosts**
  - Log writes: 10^6 × 10^3 events/s = **10^9 events/s** × 1KB = **1 TB/s**
  - **Plausibility check:** 1 TB/s is ~86 PB/day — larger than any real logging system (the big commercial pipelines run low single-digit PB/day). Either "1000 events/sec/host" is a burst figure rather than sustained, or the given is 2–3 orders of magnitude hot. I'll design for a **sustained 10^7 events/s (~10 GB/s, ~1 PB/day)** and treat 10^9 as the burst the edge must shed. *(Flagging this to the interviewer is the point — a senior does not silently design for an impossible number.)*
  - Metrics: 10^7 series ÷ 10s = **10^6 points/s**; at ~16 B/point compressed that's **16 MB/s** — three orders of magnitude smaller than logs. Different data, different store, different problem.
  - Peak 3× → ingest tier sized for 3×10^7 events/s.
  - Reads: 500 engineers × 50 queries/day = 2.5×10^4/day ≈ **0.25 QPS**.
  - **Read:write ratio ≈ 1 : 4×10^7.** This is the headline. It is a pure write-amplification problem: optimise entirely for cheap sequential append and compression, and accept that the read path scans.
  - **Storage:** logs 10 GB/s × 10^5 s/day = **1 PB/day raw**, × 30 days = 30 PB, ~10× compression → **~3 PB**. Metrics ~500 TB/yr raw → **~50 TB** after downsampling.

> **What this buys:** the plausibility line is the whole gap — it converts a 10,000× arithmetic slip into a conversation with the interviewer about which given is wrong, which is the senior move. The read:write ratio turns the rest of the design into a single obvious conclusion instead of a series of guesses, and the durability line makes load-shedding a stated decision rather than an accident.

### Core entities

- **`LogEvent`** — `(ts, serviceId, hostId, severity, message, attrs: map<string,string>)`. No primary key: logs are append-only and never addressed individually. Partitioned by `(serviceId, hour)`.
- **`MetricSeries`** — `(seriesId [pk], name, unit, labels: map<string,string>)`. **Unique on `hash(name + sorted labels)`** — that hash is how an agent resolves a series without a round trip to the server.
- **`MetricPoint`** — `(seriesId, ts, value: double)`. Unique on `(seriesId, ts)`.
- **`LogSegment`** ← **only appears under load.** An immutable, compressed, time-bounded block of events for one `(serviceId, hour)` partition, carrying min/max timestamp and a bloom filter over hostId. It exists because at 1 PB/day you cannot maintain an index over individual events — you index *segments* and scan within them. Retention and hot→cold tiering operate on segments, not events.
- **`MetricRollup`** ← **also only appears under load.** `(seriesId, windowStart, resolution, min, max, sum, count)`. Directly forced by "downsampled after 30 days": a 1-year chart at 10s resolution is 3×10^6 points per series; 1m/1h/1d tiers bring it to a few hundred.

> **What this buys:** `value` on the metric point, obviously. Beyond that, the two load-only entities are the ones his own retention answer already required — naming them is what turns "downsample after 30 days" from an acknowledged sentence into something the design can actually do. Keys and uniqueness make the series-resolution problem visible before the API phase rather than during it.

### API design

**Ingest**

```
POST /v1/logs:batch
  Idempotency-Key: <agentId>:<batchSeq>
  { events: [ { ts, serviceId, hostId, severity, message, attrs{} } ] }   // <= 1000 events or 1MB
  -> 202 { accepted: int, dropped: int }
     429 { retryAfterMs: int }      // backpressure: the agent drops, it must not queue
     413                            // batch over limit
```
Batched, not per-event — at this write rate the batch size *is* the contract.

```
POST /v1/metrics:batch
  Idempotency-Key: <agentId>:<batchSeq>
  { samples: [ { seriesKey: { name, labels{} }, ts, value } ] }
  -> 202 { accepted: int }
```
Samples carry the series *key*, not an id, so an agent never needs a round trip to start emitting.

**Query**

```
GET /v1/logs?serviceId=&hostId=&severity=&q=&from=&to=&limit=&cursor=
  -> 200 { events: [...], nextCursor: string|null, partial: bool, scannedBytes: int }
```
`partial: true` when the query hit its scan budget — at this read:write ratio some searches genuinely cannot complete, and the client must be told rather than shown a silently truncated result. Cursor, not offset: the store is time-ordered and append-only, so a cursor is a `(segmentId, offset)` pair; an offset would mean counting from the start of seven days of data.

```
GET /v1/metrics/series?name=&label.k=v&limit=&cursor=
  -> 200 { series: [ { seriesId, name, labels } ], nextCursor }
```
Discovery — without it nothing in the system ever hands the caller a `seriesId`.

```
GET /v1/metrics/query?seriesId=&from=&to=&step=&agg=avg|max|sum|p99
  -> 200 { seriesId, resolution: "10s"|"1m"|"1h"|"1d", points: [[ts, value]] }
```
`step` bounds the payload; the returned `resolution` tells the client which rollup tier answered, so it knows the fidelity it actually got. Without `step`, a one-year query returns millions of points.

**Errors:** 202 accepted · 429 + `retryAfterMs` on backpressure · 400 on a malformed batch (reject whole, never partially parse) · 413 oversized · `partial: true` on a truncated search.
**No DELETE** — retention is policy-driven and automatic. Stated, not omitted.

> **What this buys:** he had the two hardest things right already (idempotency and cursor pagination, both unprompted). What's added is the part that was blank: request bodies that make batching explicit, `nextCursor` actually named, the `q`/`severity`/`serviceId` filters FR2 requires, a series-discovery endpoint without which the metric read is uncallable, and `step`/`resolution` so a one-year chart isn't a multi-million-point response.

---

## Feedback given

**Scores:** Requirements 2/5 · Core entities 1/5 · API design 2/5 · **Front-half readiness 2/5.**

**Requirements 2/5** — availability with its arithmetic, a query latency with a percentile, and an unprompted out-of-scope list are all real. But the headline traffic number is wrong by 10,000× (the 10,000-service multiplier was dropped), it was never sanity-checked, and three walk items — read:write ratio, storage growth, durability — are absent despite the inputs for two of them being handed over. The missing plausibility check alone caps this phase at 3; the arithmetic error and three missing items put it at 2.

**Core entities 1/5** — the three surface objects are the right three, and it took 5:07 to write twelve words that omit the `value` field on `MetricPoint`. No keys, no uniqueness, no serviceId or severity on `Log`, and neither of the two entities his own retention requirement forces.

**API design 2/5** — idempotency on every mutating endpoint and cursor pagination on every list endpoint, both volunteered without prompting. That is two standing weaknesses not firing, and it should be said plainly. Against it: no request bodies at all, no response fields at all, no error semantics, no search parameter on the search endpoint, and no way to obtain a `seriesId`. The skeleton is a senior's; the contract is empty.

**Front-half readiness 2/5** — two phases substantially incomplete at the buzzer and the API never reached a usable contract.

**Is the front half slow because he thinks slowly, or because the first pass is incomplete?** Neither, this time — and that's new. Every phase closed in one message with nothing back-filled, which is a clear improvement on the previous sitting. The problem is raw throughput on the thinking phases: 10:50 to first content, and 5:07 to produce three lines. The time is being spent deciding what to write rather than writing it.

**The one habit to change:** *after every derived headline number, write one line dividing it back down to a per-unit figure and say whether it's believable.* "100 MB/s ÷ 10,000 services = 10 KB/s per service = one log line every ten seconds" takes four seconds to write and catches a 10,000× error before it reaches the entities phase.
