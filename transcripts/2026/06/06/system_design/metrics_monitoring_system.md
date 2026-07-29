# System Design Round Transcript
**Date:** 2026-06-06
**Start Time:** 15:40
**End Time:** ~16:15 (round ended early — candidate stopped at the HLD transition)
**Duration:** ~35 minutes (incomplete — no HLD / deep dive)
**Problem:** Design a Distributed Metrics & Monitoring System

---

## Conversation Log

**Interviewer:** Design a distributed metrics & monitoring system (Prometheus/Datadog-like) — ingest metrics from a large fleet, store as time-series, query, dashboards, alerts. Start with requirements.

**Aayush:** Should we support metric aggregation or leave it out of scope?

**Interviewer:** Query-time aggregation (over time and over series) is in scope. Whether to also pre-compute rollups/downsampling is a design decision for you. Out of scope: tracing, log aggregation, alert *notification delivery* (alert *evaluation* is in scope). Now give FRs + NFRs with numbers.

**Aayush:** Is it fine to assume 100M clients daily?

**Interviewer:** That's DAU framing; metrics systems scale on ingest rate (points/sec) and active-series cardinality. Anchor on a fleet: ~100k hosts × ~1k series each, 10s scrape. Derive active series, ingest rate, storage/day. Plus retention/query-latency/availability.

**Aayush:** What does "scraped every 10s" mean?

**Interviewer:** Pull model — system polls each target every 10s; each series yields one point per interval. (Push is the alternative; pick later.) Now run the three numbers.

**Aayush:** FRs — (1) ingest metrics, (2) store as time-series, (3) queries, (4) dashboards, (5) alerts on thresholds. Out of scope — tracing, log aggregation, alert notification delivery. NFRs — (1) HA for ingestion, (2) eventual consistency adjustable with strong+eventual options, (3) durability, (4) low-latency queries p99<200ms, (5) at-least-once for alerts, (6) scale 100k hosts × 1k series, 10s scrape → ingest 10^7 points/s; 16 bytes/point → 16 TB/day.

**Interviewer:** Storage is ~13.8 TB/day (160 MB/s × 86,400), not 16 — and TS compresses ~10×. You skipped retention (the dominant storage driver). And does a metrics system actually want strong consistency on ingest, or is it over-engineering?

**Aayush:** (1) Using 10^5 s/day approximation. (2) Keep 7 days hot, then move to cold storage. (3) Eventual consistency is fine — strong consistency thought was for critical metrics needing immediate alerting, but that doesn't affect the write path.

**Interviewer:** Good — dropping strong consistency from the write path is right; "immediate alerting" is freshness, not consistency. Now core entities.

**Aayush:** (asked to update drawio with FRs/NFRs first) Then: Metric(id, hostId, timestamp, value, metricType), Host(id, name), Alert(id, metricType, createdAt, alertConditions), AlertNotification(id, alertId, createdAt).

**Interviewer:** Series vs data point — is "Metric" one point or the whole series? How do you uniquely identify a series? And where do labels/dimensions live (the source of your 100M cardinality)? You only have hostId.

**Aayush:** MetricSeries(id, hostId, metricType, dimensions), MetricPoint(metricSeriesId, timestamp, value).

**Interviewer:** Right split. Now the API — ingest, query, alert-rule, with concrete shapes.

**Aayush:** Identity from JWT. POST /metrics {hostDetails, metricValues} → 2xx. GET /queries/:id → {metricPoints:[]}. (Noted a SavedDashboardQuery entity is needed.)

**Interviewer:** Too vague. Ingest: batch shape? idempotency under retry? Query: it's ad-hoc (metric+matchers+range+aggregation+step+groupBy); response is nested (list of series, each a list of points). Alert-rule endpoint missing.

**Aayush:** Help me out, I'm blanking.

**Interviewer:** (Gave an ingest example shape as scaffold; asked him to fill query + alert-rule by analogy.)

**Aayush:** A1: POST for query — too many variable parts, query params make URL clunky. A2: list of series because there can be multiple time buckets between start and end. B1: the various conditions for the alert rule.

**Interviewer:** A2 reasoning is wrong — outer list = distinct series (one per label-combo / group-by, e.g. per host), inner list = time buckets. Restate. B still vague — give concrete alert-rule fields (condition, for_duration, evaluation interval).

**Aayush:** Inner = metric values for the time-series from one host. I give up on API design.

**Interviewer:** (Accepted; noted alert-rule + ingest idempotency left open; explained for_duration prevents transient-spike pages.) Move to HLD — ingest path, query path, alert evaluation.

**Aayush:** (asked to update drawio with entities + API first)

**Interviewer:** (Updated drawio.) Now narrate the HLD — both paths.

**Aayush:** I quit.

---

## Design Summary
**Requirements Gathered:** FRs — ingest, store time-series, query, dashboards, threshold alerts. Out of scope — tracing, logs, notification delivery. NFRs — HA ingest, eventual consistency on write path, durability, query p99<200ms, at-least-once alerts, 7-day hot retention → cold. Scale derived: 100M active series, 10^7 points/s, ~14–16 TB/day raw, ~10× TS compression.

**Core Entities:** MetricSeries(id, hostId, metricType, dimensions/labels), MetricPoint(seriesId, timestamp, value), Host, Alert/AlertRule, AlertNotification, SavedDashboardQuery (TBD).

**API:** POST /v1/metrics (batch ingest, shape left rough, no idempotency); POST /v1/query (ad-hoc: metric+matchers+range+aggregation+step+groupBy → {series:[{labels, points:[{ts,value}]}]}); GET /queries/:id (saved dashboard); POST /v1/alert-rules (left incomplete).

**High-Level Architecture:** NOT REACHED — round ended at the HLD transition.

**Key Design Decisions & Trade-offs:** Pull-vs-push left open. Dropped strong consistency from write path (good). POST over GET for query (clunky URL justification).

**Scalability & Fault Tolerance Points:** Scale numbers derived well; no architecture built to handle them (cardinality, ingest fan-in, storage tiering, query fan-out all unaddressed because HLD not reached).

**Gaps / Missed Areas:** Entire HLD and deep dive; ingest batch shape + idempotency; alert-rule contract; downsampling/rollup policy for cold data; query-latency feasibility at scale; pull-vs-push decision.

---

## Feedback Given

Round ended early (candidate disengaged at the HLD transition after blanking on API design). Feedback proportionate to what was observable.

**Strengths (real):**
- **Scale derivation from first principles** — 100M active series, 10^7 points/s, ~14–16 TB/day, unprompted and clean. Directly addresses a long-standing weakness ("cannot derive scale independently").
- **Caught own over-engineering** — proposed then dropped strong consistency on the write path, correctly separating freshness (alerting) from consistency. Senior move.
- **Entity modeling** — reached correct MetricSeries vs MetricPoint split (with labels as series identity) once pushed.

**Where it broke:**
- **API design** (normally his strongest area) is where he blanked and then disengaged. Diagnosis: not an API-skill gap but a **domain-unfamiliarity** gap — he'd never modeled a time-series query response (nested series→points) and froze without a template, which snowballed into quitting.
- Recurring pattern: routed to "update the drawio" twice at the exact moment the next thinking step was asked (deferral via diagram).

**Unblock technique given:** when blanking on an API shape, start from a concrete example of the *result the user wants* ("a few lines on a graph, each line a host, each a series of (time,value) dots") and reverse-engineer the JSON from the picture, rather than reaching for the perfect endpoint. He reached the right nesting (A2) once he slowed down — he just needed to start from the picture.

**Senior-signal scorecard (partial — HLD/deep-dive not observed):**
- Own the narrative / self-raise traps — Mixed (caught consistency over-engineering himself; needed prompting on labels/series identity).
- Lead with trade-offs vs alternatives — Mixed (POST-vs-GET justified; pull-vs-push deferred and never decided).
- Push scale until it breaks — Not observed (HLD not reached).
- API as a designed contract — Weak this round (blanked on query/alert shapes; usually Strong — treat as an off day / domain gap).
- Operability / second-order concerns — Not observed.
- Pace — Weak (35 min and only through API; two diagram-deferrals; disengaged before the core).

**Overall read:** Incomplete — not gradeable as hire/no-hire. Strong opening (requirements + scale + entities at a good level), then a domain-unfamiliarity stall on the time-series query contract that ended the round before any architecture. The fix is exposure (study one TSDB query/response model — Prometheus/PromQL or Datadog) plus the "describe the result first" unblock habit, not a fundamental design-skill gap.

**What a senior strong-hire would have done on THIS problem (for later study):**
- Ingest: agent/collector → load-balanced ingest tier → Kafka (buffer/backpressure) → consumers → TSDB; batch + idempotent writes keyed on (seriesId, timestamp).
- Storage: dedicated TSDB (Prometheus TSDB / Cortex / VictoriaMetrics / InfluxDB) with Gorilla-style compression; hot SSD for recent + object-store cold tier; **downsampling/rollups** (10s → 1m → 1h) for aged data instead of raw cold dump.
- Cardinality: an inverted index from labels → seriesIds; treat high-cardinality labels as the "hot key" failure mode (the scale-break a senior would self-raise).
- Query: query layer fans out across shards/time ranges, merges, aggregates; p99<200ms only achievable on recent/pre-aggregated data — a senior would call out that arbitrary 6h raw scans cannot hit 200ms without rollups.
- Alerting: rule evaluator periodically runs each rule's query on the freshest data; for_duration state machine; at-least-once hand-off to notification.

**Time Taken: ~35 minutes (incomplete round).**

Pointed candidate to `system_design_senior_guidance.md` for the next full round.
