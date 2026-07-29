# System Design Round Transcript
**Date:** 2026-07-28
**Start Time:** 15:54:02
**End Time:** 16:39:27
**Duration:** 45.4 minutes
**Problem:** Design a Usage-Based Metering & Billing System
**Performance Rating:** 3/5

## Phase Timings
| Phase | Budget | Actual | Hit? |
|---|---|---|---|
| Requirements | 8 min | 17.2 min | No |
| Core entities | 12 min | 22.3 min | No |
| API design | 17 min | 28.8 min | No |
| High-level design | 27 min | 39.3 min | No |
| Deep dive | 40 min | 45.4 min | No |

---

## Problem Statement

Design the metering and billing platform for a B2B SaaS/cloud company. Customers are charged based on what they actually consume — API calls, compute-seconds, GB stored, seats active — rather than a flat subscription.

The platform needs to:
- Ingest usage events emitted continuously by dozens of product services
- Aggregate usage per customer, per meter, per billing period
- Apply pricing plans to produce an invoice at the end of each period
- Let customers see their current spend before the bill arrives

The bill has to be right. Over-charging is a support escalation and a refund; under-charging is lost revenue.

Interviewer-supplied facts:
- Billing is monthly, **anniversary-aligned** (signup on the 17th → bills on the 17th); period boundaries spread across every day of the month
- Single currency, no mid-period plan changes
- ~50,000 paying business customers, heavily skewed — top 1% is ~50% of volume
- ~20 metered products
- Largest single customer can burst to tens of thousands of billable actions/sec

---

## Conversation Log

**Interviewer:** [15:54:02] Presented the problem, 45-minute budget, canvas path given. Start with requirements — functional and non-functional, with numbers.

**Aayush:** [15:59:32] what are the supported billing periods?

**Interviewer:** Monthly, but anniversary-aligned — a customer who signed up on the 17th bills on the 17th. So period boundaries are spread across every day of the month, not a single month-end batch. Assume a single currency and no mid-period plan changes for now. Keep going — give me the functional requirements and the NFR numbers.

**Aayush:** [16:02:02] what scale can i assume for this system

**Interviewer:** This is yours to propose — sizing is part of the design. Business facts: ~50,000 paying business customers, heavily skewed (top 1% are ~50% of volume); ~20 metered products; largest single customer can burst to tens of thousands of billable actions per second. Derive events/sec average and peak, storage per billing period, retention, and the freshness/accuracy SLAs. We're 8 minutes in with no requirements — that's the whole budget.

**Aayush:** [16:06:55]
FRs:
1. Ingest usage events of different types (API call, compute seconds, GB stored etc)
2. Aggregate usage per customer per type, per billing period (monthly, anniversary-aligned)
3. Apply pricing plans to produce an invoice for each billing period. No overpricing or underpricing
4. Show current spend in UI.
NFRs:
1. Highly available for event ingestion (99.99% ~ 52 mins downtime/year)
2. No undercounting or overcounting of consumption events.
3. Acceptable delay between consumption event and price UI update < 3 min.
4. Durability of consumption events.

**Interviewer:** FRs are solid. But you gave an availability target and a freshness target and stopped — no throughput, no storage, no retention. Sixty seconds: peak events/sec, storage per billing period, how long you keep raw events. With the arithmetic.

**Aayush:** [16:10:19] assuming 50k customers and each customer producing 10k events/day -> 5B events/day -> 50k events/s (assuming 10^5 s in a day). assuming each event is 1kb. Total storage for 1 billing period (5 TB/day * 30) = 150TB. Raw events retained only for one billing period because of high cost of storage

**Interviewer:** Recheck that first multiply: 50,000 × 10,000 = ? You wrote 5B. Everything downstream inherits it. Also — top 1% are half the volume and can burst to tens of thousands/sec. You've given me an average and no peak. Fix the number, give me a peak multiplier, then move into core entities — we're at 16 minutes and entities were due at 12.

**Aayush:** [16:11:10] 500M events/day, 5k events/s avg and peak is 10x

**Interviewer:** [rendered FRs + NFRs to canvas] Core entities now — we're at 17 minutes and API design was due here.

**Aayush:** [16:16:18]
Core Entities:
1. Consumption event (type, id, customerId, time, quantity)
2. Customer (id)
3. BillingPeriod (type, customerId, billingStartDate, currentBillingCycleCount)
4. Invoice (id, amount, type, customerId, billingCycleCount)
5. ConsumptionType Enum (API | Compute | Storage ..)
6. PricingPlan (type, rate)

**Interviewer:** Two things, then move. Nothing in that list holds a running total — the spend UI has a 3-minute freshness SLA, which entity does it read? And `ConsumptionEvent` has an `id`. Who generates it, and what is it for? Answer both briefly, then give me the API design — we're at 22 minutes and it was due at 17.

**Aayush:** [16:17:48] another table to track the cumulative price change for a billingCycle, which would be updated upon ingesting each consumption event for a particular type and customer

**Interviewer:** You skipped the second question. Moving on — I'll come back to it. API design now. Endpoints, verbs, request and response shapes with explicit fields. Include the ingestion contract and the read paths.

**Aayush:** [16:22:46]
NOTE: CustomerId extracted from auth header.
POST /consumptionEvent/type — Idempotency key header to prevent excessive counting. Request: {consumptionQuantity, timestamp}. Response: 2xx status indicating consumption event accepted
GET /consumptionEvent/type/currentSpend — Response: Spend in currency set

**Interviewer:** Good — you raised idempotency yourself, unprompted. Two gaps, answer both in one message then go straight into the HLD: (1) "Spend in currency set" is not a response shape — give me the actual JSON fields. (2) You're taking 50k events/sec at peak as individual HTTP POSTs, one per event. Is that the contract you want?

**Aayush:** [16:27:23] 1. { priceSpend, currency}  2. Batch ingestion endpoint as well

**Interviewer:** HLD now — this is the last big block before we're out of time. Components, datastores, and the flow from event ingest through to invoice.

**Aayush:** [16:33:18] the customers trigger the post endpoints and the request is passed through an API gateway. The request is handled by the ingestion service which will write the data to a write optimised DB like dynamo DB. The ingestion service will write the event and also update the current spend pricing for the event type and customer depending on the pricing plan. The user can query the current spend via the GET request and the system will query the current spend from the DB. There will be a recurring cron which will run at the end of billingCycle for each customer and event type. This will generate the invoice based on the recorded consumption in the DB. This is a relatively simple design and it has many areas of failure such as -> high load of incoming events and ensuring no event is lost or double counted. There is also no storage management and the database can grow unbounded

**Interviewer:** Good — you named the failure modes yourself rather than waiting for me. Deep dive: your ingestion service does two writes — append the event, and update the customer's running spend total. Money has to be exactly right. What happens when the service crashes between those two writes? And what does your idempotency key actually protect, given the retry will come from the client?

**Aayush:** [16:35:37] if service crashes between writes there is chance that the event will be lost and it wont be counted. Currently the idempotency key which will be passed from customer client can be set as the id for the consumer event and uniqueness can be enforced to dedupe properly.

**Interviewer:** Sharper case: the event write succeeds, then the service dies before updating the total. Client retries with the same idempotency key. Your uniqueness constraint rejects it as a duplicate. What's the running total now, and does it ever get fixed?

**Aayush:** [16:39:27] no, in order to fix this kind of problem we need to ensure that event is not lost — for that we introduce kafka for durability of events and then have a redis cache to dedup the events before inserting them into kafka. Kafka provides at-least-once delivery guarantee and the consumers must be idempotent. So now we can accumulate the events in time windows of 1 minute, and then write the accumulated event data for the time period of 1 min to DB. Now we would be storing the consumed quantity per minute in the DB and not the running price. In order to get the running price we will just sum the quantities for the billing cycle so far and use the pricing plan to get the price. So complexity will be moved to read time and invoicing time.

**Interviewer:** Round ended at 45.4 minutes. Moved to feedback.

---

## Design Summary

**Requirements Gathered:**
- FRs: ingest typed usage events; aggregate per customer/type/period; apply pricing plans to produce invoices with no over/under-pricing; show current spend in UI.
- NFRs: 99.99% ingestion availability; no under/over-counting; <3 min event→spend-UI freshness; event durability.
- Scale (after correction): 500M events/day, ~5k events/s avg, 10× peak = ~50k events/s, 1 KB/event, raw retained one billing period. Storage figure never restated after the arithmetic correction (canvas shows the stale 150 TB; correct is ~15 TB/period).

**High-Level Architecture (v1):**
Clients → API Gateway → Ingestion Service → DynamoDB (write-optimised). Ingestion Service performs two writes: append the consumption event, and update the current spend for (customer, type) using the pricing plan. GET current spend reads from the same DB. A recurring cron runs at each customer's billing-cycle end and generates the invoice from recorded consumption.

**Deep Dive (revised ingest path):**
Clients → API Gateway → Ingestion Service → Redis (dedupe) → Kafka (durability, at-least-once) → idempotent Consumer doing 1-minute window aggregation → DynamoDB storing **consumed quantity per minute**, not a running price. Current spend and invoices computed by summing quantities over the billing cycle and applying the pricing plan at read/invoice time.

**Key Design Decisions & Trade-offs:**
- DynamoDB chosen as "write-optimised" — no alternative named, no trade-off stated.
- Kafka introduced for durability + at-least-once — no alternative named.
- Redis chosen for dedupe ahead of Kafka — durability implications not discussed.
- Deliberate shift of complexity from write time to read/invoice time (stated explicitly) — the strongest decision of the round.
- Idempotency key on the write endpoint, raised unprompted.

**Scalability & Fault Tolerance Points Covered:**
- Identified event loss and double-counting as the core risks in his own v1 design, unprompted.
- Identified unbounded database growth, unprompted.
- Correctly diagnosed that the two-write problem is solved by eliminating the second write rather than making the pair atomic.
- Accepted a 10× peak multiplier when prompted.

**Gaps / Missed Areas:**
- Hot tenants / hot partitions — the top-1%-is-50%-of-volume skew was given in the prompt and never used; no partitioning scheme at all.
- Tiered pricing — `PricingPlan (type, rate)` assumes price is linear in quantity; `sum(price) ≠ price(sum)` under tiers.
- Read-side scale break in his own final design: 1-min buckets = 43,200 rows/customer/meter/period, ~864k rows summed per current-spend query across 20 meters. No rollup hierarchy.
- Late / out-of-order events arriving after invoice finalization — no policy, no grace window, no period state machine.
- Redis dedupe durability — a Redis loss admits duplicates, i.e. overcharging, violating a stated NFR.
- No invoice endpoints, no pagination, no batch-endpoint shape, no `asOf` on the spend response.
- Zero operability: no monitoring, no consumer-lag alerting, no DLQ, no reconciliation job, no invoice-delta alerts.
- Invoice not reproducible — no pricing-plan version or aggregate snapshot stored.
- No load balancer; no read/write path separation; raw events in DynamoDB rather than object storage despite flagging storage cost.
- Storage number left uncorrected.

---

## Feedback Given

# Round Complete — 15:54:02 → 16:39:27 · **Time Taken: 45.4 minutes**

That last answer was the strongest thing you did all round. Removing the running-total write instead of trying to make two writes atomic is the correct instinct, and you got there yourself after one pointed question.

## Feedback

**Requirements clarification — 2/5**
You opened with two real clarifying questions, which is an improvement. But you asked *me* for the scale numbers — that's yours to propose. When you did produce them, `50,000 × 10,000 = 5B` is off by 10×, in a system whose entire premise is that the arithmetic has to be right. You corrected it when challenged, but never restated storage, which still reads 150 TB on your canvas. It's **~500 GB/day → ~15 TB per period**.

You also gave no peak until I asked, despite my telling you in the problem statement that the top 1% are half the volume and burst to tens of thousands/sec. That skew is the single most important fact about this system and it never entered your design.

Requirements consumed **17 minutes of an 8-minute budget**, and that is why the deep dive got six.

**Core entities — 3/5**
The list was reasonable and you were right to separate `PricingPlan` from `ConsumptionEvent`. But the entity the whole system exists to produce — the per-customer, per-meter, per-period aggregate — was missing until I pointed at it, and then arrived as "another table" rather than a modelled entity. `PricingPlan (type, rate)` is a flat rate; real usage billing is tiered, and that single field choice quietly assumes price is linear in quantity.

**API design — 2/5**
Two endpoints and a batch endpoint you named but never specified. No invoice endpoints at all — no `GET /invoices`, no way to fetch a specific invoice, no pagination, despite invoices being an inherently listable, growing resource. `{ priceSpend, currency }` came only after I rejected "spend in currency set," and it still has no period boundaries, no per-meter breakdown, and no `asOf` timestamp — which a 3-minute-stale number absolutely needs.

The one strong move: you put the idempotency key on the write endpoint **unprompted**. That's a senior habit and it's the first time it's shown up.

**High-level architecture — 3/5**
The v1 design was coherent and you reached it quickly once you started. Critically, you closed it by naming its own failure modes — event loss, double counting, unbounded storage — without being asked. That is genuinely the right posture and it's new.

But it arrived at **39 minutes**, and a synchronous "update the running total on the write path" design at 50k events/sec is a first draft, not a design. Which you knew.

**Component design & trade-offs — 1/5**
You named DynamoDB, Kafka, and Redis. You justified **none** of them against an alternative. Why DynamoDB and not Cassandra, or S3 + a columnar store, or Postgres with partitioning? Why Kafka over Kinesis or SQS? What does Redis-for-dedupe cost you? "Write-optimised DB like DynamoDB" is a label, not a decision.

This is now 15 sessions of the same gap and it is the cheapest point on the rubric to fix: every box gets *"…over X, because Y, and I'm giving up Z."*

**Scalability & fault tolerance — 3/5**
Strong finish, weak middle. The Kafka-for-durability + idempotent-consumer + windowed-aggregation answer is correct, and "store quantity, not price, and compute at read time" is exactly right. But you never pushed scale on your own — I supplied the 10×, I supplied the crash scenario, and the hot-tenant skew I handed you in the prompt was never used.

**Deep dive quality — 3/5**
Compressed into six minutes, and within that constraint it went naive → break → fix properly. You correctly identified that the fix isn't making two writes atomic but eliminating the second write. What's missing is everything after the first fix: no trade-offs stated on the new design, no discussion of what Redis-based dedupe costs you, no late-event handling.

**Communication — 3/5**
Clear and structured when you commit to an answer. But you answered one of two questions three separate times this round — the event-`id` question, the response-shape/batch pair, the entity/id pair.

**Diagram quality — n/a (interviewer held the pen).** For the record: the described design had no load balancer, no DLQ, no monitoring, and no separation between the read path and the write path — the same service and store served ingestion, spend queries, and invoicing.

**Pace**

| Phase | Budget | Actual | Hit? |
|---|---|---|---|
| Requirements | 8 min | 17.2 min | **No** |
| Core entities | 12 min | 22.3 min | **No** |
| API design | 17 min | 28.8 min | **No** |
| High-level design | 27 min | 39.3 min | **No** |
| Deep dive | 40 min | 45.4 min | **No** |

Every checkpoint missed. The one real positive: you finished at **45 minutes** rather than the 60–70 you've historically run to. But you did that by having almost no deep dive, which is where the senior signal lives.

---

## Senior Readiness Debrief

### Senior-signal scorecard

| Signal | Status | Why |
|---|---|---|
| Owns the narrative / self-raises traps | **Mixed** | Self-raised idempotency and self-critiqued the v1 HLD unprompted — real progress. But peak load, the aggregate entity, batch ingest, and response shapes all needed prompting. |
| Leads with trade-offs vs named alternatives | **Weak** | Zero alternatives named for any component, all round. |
| Pushes scale until it breaks | **Weak** | Interviewer supplied the 10×, the crash scenario, and the tenant skew. The skew was never used. |
| API as a designed contract | **Weak** | 2.5 endpoints, no invoice resource, no pagination, thin shapes. Idempotency key is the one bright spot. |
| Operability / second-order concerns | **Weak** | Nothing at all: no monitoring, no DLQ, no consumer lag, no late events, no reconciliation. In a billing system this *is* the job. |
| Pace | **Weak** | Every phase over; HLD at 39 min; deep dive starved to six. |

**Level read:** mid-level. At a senior bar this is a **no-hire**; at a mid bar it's a **hire**. The technical instincts in the last five minutes were senior-grade — the process around them wasn't.

### **Performance Rating: 3/5 — Pass.** Eligible for re-ask.

---

## What a senior strong-hire would have done on THIS problem

**1. Used the tenant skew.** "Top 1% are 50% of volume" was in the prompt. That means partitioning by `customerId` puts a single tenant's entire 25k events/sec on one Dynamo partition — a guaranteed hot partition. The fix is a composite key `customerId#shard` with `shard ∈ [0, N)` where `N` scales with tenant size, and a fan-in read. No partitioning was discussed at all.

**2. Named the reason to store quantity instead of price.** The right answer was reached for the right general reason (idempotency), but the *stronger* reason is pricing itself: real usage plans are **tiered** — first 1M calls at $0.001, next 10M at $0.0008. Price is not linear in quantity, so `sum(price)` ≠ `price(sum)`. Once tiering exists, storing a running price is not just fragile, it's **arithmetically wrong**. `PricingPlan (type, rate)` assumed it away.

**3. Found the scale break in their own final design.** The revised design stores one row per (customer, meter, minute). That's 1,440/day × 30 = **43,200 rows per customer per meter**, × 20 meters = **~864,000 rows** that a current-spend query must sum. At a 3-minute freshness SLA and a UI page load, that read does not work for the largest tenants.

The senior fix is **hierarchical rollups**: minute buckets roll into hour buckets roll into day buckets. Current spend = sum ~30 day-rows + ~24 hour-rows + ~60 minute-rows ≈ 115 reads instead of 864,000. Complexity was correctly moved to read time — but never sized.

**4. Raised late and out-of-order events.** The hardest real problem in billing, and it never came up. An event with a timestamp inside a period arrives *after* the invoice was generated. Re-bill? Credit note on the next invoice? Hold finalization for a grace window? A senior states a policy: **invoices finalize after a 24-hour grace window; anything later becomes an adjustment line on the next period** — and makes the period a state machine (`open → closing → finalized`) so "finalized" is immutable and reproducible.

This is sharper here because billing is **anniversary-aligned** — given in the first answer. Period boundaries land on all 28+ days of the month, so finalization is a continuous background process, not a month-end batch. A cron was proposed with no discussion of what happens when it fails, runs twice, or runs while events are still landing.

**5. Noticed Redis dedupe is not durable.** Dedupe sits in Redis in front of Kafka. If Redis loses its set — eviction, failover, restart — duplicates flow through and customers get **overcharged**, violating a stated NFR. A senior names this immediately and either moves dedupe into the sink (idempotent upsert keyed on `eventId` within the minute bucket) or uses Kafka transactions / a stream processor with exactly-once sinks. Redis dedupe is a *latency* optimisation in front of a durable check, not the check itself.

**6. Owned monitoring — "how do I know the bill is wrong?"** Nothing in the design would surface a break. A senior specifies: consumer lag alerting against the 3-minute SLA; daily reconciliation of raw event count vs sum of aggregate rows, alerting on drift; per-customer invoice-delta alerts (this bill is 10× last period → hold for review before it reaches the customer); and a replayable raw-event log so any invoice can be recomputed from source. In billing, the reconciliation job is a **first-class component**.

**7. Made the invoice reproducible.** Store the plan **version** and an immutable aggregate snapshot alongside the invoice, so a dispute six months later can be answered exactly. `Invoice (id, amount, type, customerId, billingCycleCount)` cannot be re-derived if the pricing plan has since changed.

**8. Put raw events in the right store.** 15 TB/period at 50k writes/sec in DynamoDB is extremely expensive. Raw events belong in object storage (S3) partitioned by `customer/date`, with only aggregates in a low-latency store. Unbounded storage growth was flagged as a weakness — the trade-off discussion was right there and wasn't taken.

---

**Drill before the next round:** open `system_design_senior_guidance.md` and run the pre-round self-check. Sections 2 (trade-offs vs named alternatives) and 5 (operability) are where the most was lost today, and both are pure discipline — no new knowledge required. Specifically: force the sentence *"…over [named alternative], because […], and I'm giving up […]"* for every box drawn, and reserve the last five minutes of every round for monitoring, hot keys, and failure handling before being asked.
