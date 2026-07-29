# System Design Round Transcript
**Date:** 2026-06-10
**Start Time:** ~9:52 (not explicitly recorded — estimated from prior round end)
**End Time:** 10:47
**Duration:** ~55 minutes
**Problem:** Design a Webhook Delivery System

---

## Conversation Log

**Interviewer:** Introduced the problem — a platform (Stripe/GitHub/Shopify-style) must reliably deliver event payloads to thousands of third-party customer HTTP endpoints. Asked Aayush to drive requirements (FRs + NFRs with numbers).

**Aayush:** Customers subscribe for events they wish to be notified about?

**Interviewer:** Yes — customers register endpoint URLs and subscribe to specific event types; on an event we deliver to every subscribed endpoint. Pushed him to propose requirements.

**Aayush:** Do we need to support multiple priority levels for the webhook notifications?

**Interviewer:** No priority levels for v1. Pushed him to stop asking and start proposing FRs/NFRs with numbers.

**Aayush:** Is it fine to assume scale of 10k event notifications/s and 100k at peak?

**Interviewer:** Accepted. Pushed for fan-out, delivery guarantee, latency, retention, and explicit FRs.

**Aayush:** FRs: (1) Customers subscribe to events (produced by our system); (2) Customers register their own URL to receive notifications. NFRs: (1) at-least-once delivery; (2) low latency <1s; (3) 99.99% availability; (4) 10k/s avg, 100k/s peak. Asked interviewer to help derive fan-out estimates.

**Interviewer:** Pushed the fan-out derivation back to him — reason about distinct subscribers per event type.

**Aayush:** Assume fan-out of 5 subscribers → 50k deliveries/s avg, 500k/s peak.

**Interviewer:** Accepted, locked requirements, rendered FRs/NFRs. Moved to core entities.

**Aayush:** Core Entities: (1) Event (id, type, status, details); (2) EventSubscriber (id, clientId, eventType, eventStatus); (3) Client (id, redirectUrl). Will evolve as design progresses.

**Interviewer:** Probed: where does per-delivery state live when one of 5 endpoints is down and retried?

**Aayush:** Need to add a DeliveryAttempt entity to record delivery status per subscriber per event. Fields: DeliveryAttempt (id, status, retryCount, attemptLogs, clientId, eventId, lastAttemptAt).

**Interviewer:** Rendered entities. Moved to API design — asked for concrete endpoints, verbs, request/response shapes, including how events enter the system.

**Aayush:** (identity from auth headers) POST /clients {clientName, redirectUrl} → 2xx; PUT /clients/:id/subscribe {eventDetails:[{eventType, eventStatus}]} → EventSubscription(...).

**Interviewer:** Flagged three holes: (1) how do events enter the system; (2) no read endpoint for delivery status; (3) no idempotency protection on POST /clients.

**Aayush:** Events enter via an internal queue, no public ingestion API. Added Idempotency-Key header to POST /clients. Added GET /events/:id/deliveryAttempts?clientId={} → DeliveryAttempt(...). Argued no pagination needed since bounded by fan-out.

**Interviewer:** Accepted (idempotency key correct, internal-queue ingestion clean, per-event read is fan-out-bounded). Rendered API. Moved to HLD — trace event from queue to successful POST.

**Aayush:** Internal services write events to Kafka (durable, at-least-once, so consumers must be idempotent; producers can include idempotency keys). Stateless workers pull events, fetch subscribers + redirect URLs, POST payload via HTTP, and write DeliveryAttempt records. Possibility of duplicate delivery remains; workers generate their own idempotency keys in request headers so clients can dedupe. SQL DB for storage (also good for querying delivery attempts). Scale via more Kafka partitions + horizontally scaled stateless workers. Each event = at most 5 HTTP calls, "not very much."

**Interviewer:** Rendered HLD faithfully. Probed the deep dive: a customer endpoint times out (30s hangs) with thousands of events backed up behind it — what is the worker doing, what happens to other events in the partition, how/when does retry happen?

**Aayush:** Need timeouts on HTTP POST. On failure, worker puts event back on the queue; another worker picks it up, checks DB delivery-attempt records to see which clients are already delivered, retries the rest. On repeated failure → DLQ. Track consumer lag (to decide scaling) and client endpoint failure rate.

**Interviewer:** Pushed on two points: (1) endpoint down for an hour → immediate re-queue = tight retry-storm loop; Kafka has no native delayed redelivery, so how to space retries with backoff; (2) DLQ after only 2 failures too aggressive.

**Aayush:** Store nextRetryTime in the delivery attempt record. Worker sees event, checks nextRetryTime; if currentTime < nextRetryTime, put event back in queue, else retry. Retry policy: 3 spaced failures → DLQ; configurable; could let clients register their own retry config.

**Interviewer:** Pointed out the re-queue + clock-check is a busy-wait spin loop — the event cycles through the queue continuously for 5 min, burning Kafka throughput and worker capacity needed for 50k fresh events/s. Asked where a not-yet-due retry should actually live so nobody touches it until due.

**Aayush:** A simpler alternative: a background process that queries delivery-attempt records in the DB and puts due events into Kafka. Would poll at fine-grained intervals.

**Interviewer:** Affirmed — scheduled poller breaks the spin loop. Two follow-ups: (1) make the poller query cheap at scale (avoid full table scan); (2) prevent two poller instances from double-injecting the same retry.

**Aayush:** Indices on status + retryTime make the query cheap. For double-injection: system is at-least-once anyway, so 2 webhooks is acceptable — client dedupes via the idempotency key in the webhook body. Accept this trade-off to reduce complexity.

**Interviewer:** Accepted as a well-reasoned trade-off; noted cheaper mitigations exist (SELECT ... FOR UPDATE SKIP LOCKED, or sharding poller by time-bucket). Wrapped up.

---

## Design Summary

**Requirements Gathered:**
- FRs: subscribe to event types; register endpoint URLs; (implicitly) deliver with retries; query delivery status.
- NFRs: at-least-once delivery; <1s latency; 99.99% availability; 10k/s avg, 100k/s peak events; fan-out 5 → 50k/s avg, 500k/s peak deliveries.

**High-Level Architecture:**
Internal services → Kafka (durable, partitioned, at-least-once) → stateless horizontally-scaled workers → SQL DB (clients/URLs, subscribers, delivery attempts) → HTTP POST to customer endpoints. Idempotency keys at both producer level and on outbound webhooks for client dedup. Retries via DB nextRetryTime + scheduled background poller re-injecting due events into Kafka. DLQ after configured retry exhaustion. Metrics: consumer lag, endpoint failure rate.

**Key Design Decisions & Trade-offs:**
- Kafka for durability + at-least-once; consumers idempotent.
- Internal queue for ingestion, no public event API.
- Idempotency key on POST /clients and on outbound webhooks.
- DB-backed scheduled poller for delayed retries (over busy-requeue spin loop).
- Accept occasional double-delivery (rely on at-least-once + idempotency contract) over exactly-once complexity.
- Configurable / client-registered retry policy.

**Scalability & Fault Tolerance Points:**
- More Kafka partitions + stateless workers for throughput.
- HTTP timeouts to free worker threads.
- DLQ for exhausted retries.
- Index on (status, nextRetryTime) for cheap poller query.
- Consumer lag + endpoint failure-rate monitoring.

**Gaps / Missed Areas:**
- Head-of-line blocking from slow endpoints not self-identified (claimed "5 calls not much").
- No per-customer fairness / tenant isolation — a noisy neighbor with a dead endpoint starves others.
- No circuit breaker for consistently-failing (poisoned) endpoints.
- Delivery ordering not considered (out-of-order under parallel at-least-once).
- Retry-scheduling alternatives (delay topics, Redis ZSET) not named.
- Components stated without naming rejected alternatives.
- Pace ran long; no start clock set.

---

## Feedback Given

### Requirements Clarification — 3.5/5
Good clarifying questions and clean numeric NFRs, but opened by asking questions rather than proposing, and outsourced the fan-out math to the interviewer. FRs missed delivery+retry as an explicit requirement.

### Core Entities — 4/5
Listed Event, EventSubscriber, Client; added the pivotal DeliveryAttempt once probed, with sensible fields. Didn't volunteer DeliveryAttempt initially.

### API Design — 4/5
Strong: idempotency key on POST /clients, internal-queue ingestion, correct reasoning that per-event read is fan-out-bounded (no pagination). Gap: PUT vs POST on subscribe; response shapes needed prompting.

### High-Level Architecture — 4.5/5
Strongest phase. Clean end-to-end trace; self-raised idempotency-under-at-least-once and client dedup header; clear scaling story.

### Component Design & Trade-offs — 3.5/5
Consciously owned the double-delivery trade-off; but several choices without named alternatives, and first retry mechanism was a busy-wait spin loop caught only on prompting.

### Scalability & Fault Tolerance — 4/5
Timeouts, DLQ, lag + failure-rate metrics self-raised. Retry-storm/backoff not anticipated but solved well once surfaced.

### Deep Dive Quality — 4/5
Naive → broken (spin loop) → correct fix (DB-backed scheduled poller) with sound follow-ups (index, accept dups). Needed the break handed to him but the fix was genuinely good.

### Communication — 4/5
Clear and structured; still some fragmented ideas needing prompts, but tighter than average.

### Overall: ~3.9/5 — Solid hire-level round, edging toward senior in the deep dive.

---

## Senior Readiness Debrief

### Senior-Signal Scorecard
| Signal | Status | Reason |
|---|---|---|
| Own the narrative / self-raise traps | Mixed | Self-raised idempotency + dedup header; needed the retry-storm and spin-loop handed to him. |
| Lead with trade-offs vs named alternatives | Mixed | Owned the double-delivery trade-off; named no alternative for SQL/Kafka/retry mechanism. |
| Push scale until it breaks | Weak | Sized comfortable case ("5 calls, not much") and stopped; the break came from the interviewer. |
| API as a designed contract | Mixed | Idempotency key + internal-queue ingestion sharp; verbs and response shapes needed prompting. |
| Operability / second-order concerns | Mixed→Strong | Self-raised lag, failure-rate, DLQ; missed fairness/isolation and poisoned-endpoint detection. |
| Pace | Weak | ~55 min, no start clock, long early clarifying; core HLD started past midpoint. |

**Level read:** Hire (mid-to-senior), not yet strong-hire. Deep-dive recovery was senior-grade; held back by sharp edges being extracted rather than driven, and long pace.

### What a senior strong-hire would have done on THIS problem
1. Self-raised head-of-line blocking the moment "5 synchronous HTTP calls" came up — decouple delivery from consumption, per-target retry.
2. Named the retry-scheduling menu: Kafka tiered delay topics / Redis ZSET by nextRetryTime / DB poller — and justified the pick.
3. Driven to the noisy-neighbor / hot-partition break: one customer's dead endpoint floods retries and starves others → per-tenant fairness, isolated queues/rate-limits, circuit-breaking, capacity caps. The genuinely hard part, never reached.
4. Operability: circuit breaker for poisoned endpoints (stop hammering, alert customer); delivery ordering under at-least-once + parallel workers.
5. Pace: requirements+entities+API in ~15 min, leaving 30+ for the fairness/circuit-breaker deep dive.

**Drill:** self-check against system_design_senior_guidance.md six-point list, especially #1 (self-raise the trap) and #3 (push scale until it breaks).

**Time Taken: ~55 minutes**
