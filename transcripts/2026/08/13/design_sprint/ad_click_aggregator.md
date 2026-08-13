# Design Sprint Transcript (front half, timeboxed)
**Date:** 2026-08-13
**Start Time:** 12:20:55 · **End Time:** 12:36:41
**Problem:** Ad Click Aggregator (near-real-time click metrics, billing downstream)
**Front-half readiness: 2/5**

| Phase | Cutoff | Landed at | Cut off? | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 8:00 | never (empty at gate) | Yes | 2 (both clarifying questions, 0 content) | 0/5 |
| Core entities | 12:00 | 11:33 | No | 1 (+1 post-buzzer requirements message) | 3/5 |
| API design | 17:00 | 15:23 | No | 1 | 3/5 |

**First-pass completeness:** The requirements box produced nothing; the content arrived at 9:12, after the buzzer, and was reasonable — FRs, four NFRs, a traffic model. That message is exactly what message one should have been. The clarifying question at 8:55 ("what time windows for the dashboards?") re-asked what had already been answered at 1:43 ("minute granularity, queryable up to 1 year back"). Entities and API were single-pass and on time, but both first passes dropped the dimension fields (country/device) that the FRs explicitly name.

**Plausibility check:** Not performed. "10M customers, each ad clicked 10 times a day -> 1000 clicks/s" — the arithmetic is also off (100M events/day ÷ 86.4k ≈ 1160/s), and the model confuses advertisers with clicks. A one-line check against reality (a real ad network serves billions of impressions/day; 1k clicks/s is 2–3 orders of magnitude low) would have caught it and changed the entire ingest design. Caps Requirements independently of the gate miss.

## What he produced (verbatim, as it stood at each gate)

### Requirements
**At the 8:00 buzzer: empty.** Two clarifying questions only:
```
what time window should be considered ?          (asked at 1:43)
what time windows for the dashboards ?           (asked at 8:55 — post-buzzer)
```

Submitted at 9:12, **after the gate, scored as missing**:
```
FRs ->
1. Ingest ad-click events from web
2. Store near real time aggregated metrics - clicks per ad, per campaign, sliced by
   time window and by dimensions like country or device. (Minute granularity
   aggregation, queryable upto 1 year)
3. Dashboards over the aggregates.
4. Ad-click events used to support billing.

Out of scope ->
   (empty)

NFRs ->
1. Highly available for event ingestion
2. No undercounting or overcounting , since billing depends on counts.
3. Freshness latency of 1 min for dashboards.
4. Ad-click events stored durably for reconciliation.
5. Assuming running campaigns for 10M customer , with each ad being clicked
   10 times a day -> 1000 clicks/s
```

### Core entities
```
1. AdClickEvent (adCampaignId, timestamp, customerId)
2. ClickCountMinuteAggregate(customerId, adCampaignId, timestamp, clickCount)
3. AdCampaign (id, customerId, website)
4. ClickCountHourly/DailyAggregate(customerId, adCampaignId, timestamp, clickCount)
```

### API design
```
NOTE: Identity extracted from auth headers

1. POST /adCampaigns/:campaignId/clicks ->
Request: {clickEvents:[timestamp1,..]}
Idempotency key header

2. GET /adCampaigns/:id/aggregate?startDate={}&endDate={}&timeWindow={}&cursor={}&limit={}
Response: ClickCount(Daily/Hourly/Minute)Aggregate()[]
Cursor based pagination to allow for dynamic aggregate insertion
```

## What was missing at each buzzer

**Requirements (nothing existed at the gate):** everything. FRs, out-of-scope, all NFRs, all numbers. The 8-minute box bought two questions, the second of which was redundant with an answer already given.

**Core entities (inside gate):**
- **No dimension fields** (country, device) on any entity, though FR 2 names them — the aggregate rows cannot answer the query the API exposes.
- No dedupe/idempotency record, despite NFR 2 demanding exactly-once for billing.
- No `Ad` entity distinct from `AdCampaign`, though the problem asks for clicks-per-ad.
- No keys, partition keys, or uniqueness constraints stated.
- No raw-event storage/retention entity for the stated reconciliation requirement.
- Nothing marked as denormalised, though the hourly/daily rollups are exactly that.

**API design (inside gate):**
- Response shape is a bare type name with empty parentheses — `ClickCountDailyAggregate()[]` — zero named fields. This is the most-repeated gap in the file and it recurred here.
- No dimension filters (`country=`, `device=`) on the aggregate query despite the FR.
- Request body is a bare timestamp array; no ad id, no dimensions, no client event id.
- No error/status semantics; no 202-vs-200 distinction on an async ingest path.
- No reconciliation/backfill or delete endpoint.
- Cursor pagination justified (credit) but no `nextCursor` in the response shape to make it work.

## Where the time went
The entire requirements budget went to clarification. The first question at 1:43 was legitimate; the second at 8:55 was not — it asked for information already supplied and it landed after the gate had already closed. Everything after that ran on schedule: entities at 11:33, API at 15:23, both inside their gates with time to spare that was not used to add fields or endpoints.

## Feedback given
This was the more expensive of the two problems despite producing the better API. The content he eventually wrote was solid — exactly-once for billing, freshness SLO, durability for reconciliation are all senior-grade instincts — and all of it was worthless because it arrived after the buzzer. The single behavioural fix: write the FR and out-of-scope lists first, ask questions second, and never re-ask something already answered.
