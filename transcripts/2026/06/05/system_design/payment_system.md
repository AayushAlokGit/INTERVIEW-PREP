# System Design Round Transcript
**Date:** 2026-06-05
**Start Time:** 9:35
**End Time:** 10:47
**Duration:** 72 minutes
**Problem:** Design a Payment / Billing System

---

## Conversation Log

**Interviewer:** Design a Payment / Billing System — a service that charges customers and moves money, accepting payment requests from platforms (merchants), processing through PSPs, and keeping an accurate ledger. Start by gathering requirements.

**Aayush:** Scoping questions — Users are the platforms but actual traffic comes from the platforms' customers? Our system doesn't move money itself, just triggers the money-handling APIs? What are the details of the money-management APIs?

**Interviewer:** Correct on all three. Clients are merchant backends (server-to-server); traffic driven by their end-customers. We orchestrate via external PSPs (Stripe/Adyen-like), not the rails ourselves. PSP exposes authorize/capture/refund, is async + unreliable (timeouts, webhooks for final status, slow).

**Aayush:** Is it fine to assume 10M DAU, each making 5 payments/day?

**Interviewer:** Yes — now do the math (avg/sec, peak, storage) and give latency/consistency/availability targets.

**Aayush:**
FRs: (1) Submit (initiate & retry) payments to PSP; (2) Status tracking; (3) Accurate ledger / log of payment events.
Out of scope: fraud detection, payment analytics.
NFRs: (1) Highly available for reception (99.99%); (2) Eventually consistent status updates; (3) Low latency initiation p99<200ms; (4) Ordering + durability of events per payment; (5) Scale 10M DAU × 5/day, ~500/s avg, 10× peak.

**Interviewer:** Arithmetic: 50M/day ÷ 86,400 ≈ 580/s avg, ~5,800/s peak. Storage? And a consistency probe: merchant retries a timed-out submit under at-least-once — what dangerous property is missing?

**Aayush:** Idempotency keys needed to prevent double-charging. (Asked to add FRs/NFRs to drawio.)

**Interviewer:** Correct. Storage and peak confirmation still owed.

**Aayush:** 50GB/day. Peak calculated assuming 10^5 seconds/day.

**Interviewer:** ~18TB/year; live working set is small. Now enumerate core entities.

**Aayush:** PaymentAttempt, Payment, MerchantPlatform, PaymentEvent (attributes later).

**Interviewer:** Good — give key fields for Payment & PaymentAttempt, and where the idempotency key lives.

**Aayush:** Payment(id, merchantId, customerId, status, paymentDetails, amount, currency, createdAt); PaymentAttempt(id, paymentId, attemptLogs, pspId, createdAt, status, amount, currency). Existence of a payment record acts as idempotency dedupe; check only for new requests, not retries.

**Interviewer:** Keyed on what? New vs retry — how distinguished? And the concurrent double-submit race?

**Aayush:** Client sends an idempotency key saved on the payment record; unique constraint on that column so concurrent double-writes fail at the DB. Retries are system-owned (client sends payment details + retryConfig; system retries on PSP failures). Unique constraint solves the race.

**Interviewer:** Clean. Now design the API contract — endpoints, request/response shapes, pagination, status codes.

**Aayush:** Asked whether system should push notify merchant or let merchant poll.

**Interviewer:** Both — webhook push (happy path) + status GET (fallback/reconciliation). Now give the contract.

**Aayush:** (Asked to add push-notify FR and refund FR to drawio.) Then:
- POST /payments — Request {merchantId, amount, paymentDetails, customerId, currency, retryConfig}, Idempotency-Key header; Response {paymentId}.
- GET /payments/:id — {paymentId, amount, currency, status, attemptHistory[]}.
- GET /payments?cursor=&limit=&merchant= — cursor pagination (to include new records); Response {Payment[], nextCursor: 'paymentTs+paymentId'}.
- POST /payments/:id/refunds — {refundReason}; Response {refundPaymentId}; Idempotency-Key header; RefundPayment as its own entity.
- PUT /merchants/:id — {webhookUrl}; Response 2xx.
Identity from auth headers.

**Interviewer:** Status code on POST? Webhook payload + anti-spoof/dedupe?

**Aayush:** 202 Accepted with status pending. Webhook payload = {paymentId, paymentStatus} from PSP. (Did not address signing/dedupe.)

**Interviewer:** Now the HLD — walk the write path.

**Aayush:** Merchant → API Gateway (auth) → Payment Service creates DB record (status Queued/Pending) with client idempotency key + retryConfig → publishes payment event to Kafka (durability), partitioned by paymentId for ordering. Consumers/workers pick up events, call PSP APIs; on failure write a PaymentAttempt, follow retryConfig, push retry event to Kafka. DLQ for events exhausting retries. On success update status in DB. PSP triggers our webhook → a separate Webhook Service receives it, notifies the merchant's webhook, and updates payment status in DB. Kafka per-partition WAL serves as the ledger; extract Kafka data.

**Interviewer:** (Drew the HLD as described.) Three probes: (1) name the DB + alt; (2) the retry double-charge — a timeout is ambiguous, your logs can't disambiguate, how do you avoid a second charge?; (3) Kafka as ledger — retention/queryability/audit?

**Aayush:** (1) SQL DB for quick querying + uniqueness constraints. (2) Get reason for last retry from PaymentAttempt logs, check if retryable. (3) Transport events to a data warehouse.

**Interviewer:** Sharpened #2: your logs cannot tell charged vs not-charged — the call timed out. What do you send to / ask the PSP? (You built this at your own API edge.)

**Aayush:** Use the idempotency key with the PSP to check status for that key. DB sharded on merchantId.

**Interviewer:** Correct on #2 (idempotency key to PSP / query by key). Hot shard — what if one merchant is a whale? And #3 system-of-record store?

**Aayush:** Hot shard handled by composite shard key merchantId + paymentId.

**Interviewer:** (Noted trade-off: loses co-location → list-by-merchant becomes scatter-gather.) System-of-record store?

**Aayush:** Our own DB serving as the PaymentEvent store. (Current time 10:47.)

---

## Design Summary
**Requirements Gathered:** FRs — submit/retry payments to PSP, status tracking (pull), accurate ledger of events, push-notify merchant (webhook), refund a processed payment. Out of scope — fraud, analytics. NFRs — 99.99% availability for reception, eventually consistent status, p99<200ms initiation, ordering+durability per payment, idempotency keys for no double-charge. Scale — 10M DAU × 5/day = 50M/day ≈ 580/s avg, ~5,800/s peak, 50GB/day ≈ 18TB/yr.

**High-Level Architecture:** Merchant → API Gateway (auth) → Payment Service (writes pending Payment record with idempotency key + retryConfig; unique constraint dedupes) → Kafka (partitioned by paymentId, ordering + durability) → Consumers/Workers (call PSP authorize/capture/refund; write PaymentAttempt; retry per config; exhausted → DLQ) → PSP. PSP webhook → Webhook Service (notify merchant webhook + update DB status). SQL DB as store; PaymentEvent store as ledger; warehouse for analytics.

**Key Design Decisions & Trade-offs:** Payment vs PaymentAttempt split for clean retries/idempotency. Client-supplied idempotency key + DB unique constraint for exactly-once at API edge and concurrency race. System-owned retries. Async handoff via Kafka to keep POST under p99 SLA. Partition by paymentId for per-payment ordering. Cursor pagination (composite paymentTs+paymentId) for live data. Idempotency key forwarded to PSP to make worker retries safe under timeout ambiguity. Composite shard key (merchantId+paymentId) to spread hot merchants.

**Scalability & Fault Tolerance Points:** Kafka durability + DLQ; per-payment ordering; SQL sharding for 18TB/yr; hot-shard mitigation via composite key; idempotency at both API edge and PSP boundary.

**Gaps / Missed Areas:** PSP retry double-charge not self-raised (initially answered wrong). Webhook signing/dedupe (anti-spoof) not addressed. Outbound-webhook retry/backoff/DLQ missing. No reconciliation job for stuck/ambiguous pending payments. No status-transition state machine for out-of-order PSP webhooks. No monitoring/alerting on stuck payments, consumer lag, DLQ depth. Trade-offs frequently stated without naming the rejected alternative. DB engine not named until pushed. Pace: 72 min vs 45–50 target; repeated diagram-deferral instead of answering.

---

## Feedback Given

### Standard Feedback
- **Requirements — Strong.** Volunteered clean FR/out-of-scope/NFR split unprompted with the right NFR tensions. Minor arithmetic slip (500 vs ~580/s); needed nudge for storage.
- **Core entities — Strong.** Payment vs PaymentAttempt split was the right instinct and paid off all round.
- **API design — Strong (best phase).** Composite cursor with correct justification, idempotency headers on writes, refund as resource+entity, 202+pending for async.
- **HLD — Strong.** Async Kafka handoff, partition-by-paymentId ordering, DLQ, dedicated webhook ingest.
- **Component design & trade-offs — Mixed.** Good choices, rarely names rejected alternative (SQL engine, shard key).
- **Scalability & fault tolerance — Mixed.** Had DLQ; every scale question (sharding, hot shard) had to be driven by interviewer.
- **Deep dive — Mixed.** PSP retry double-charge — first answer didn't engage the trap; reached the right answer (idempotency key to PSP) only after two sharpenings.
- **Communication — Mixed.** Clear, but routed to "update the drawio" three times instead of answering; ran 72 min vs 45–50.

### Senior Readiness Debrief

**Senior-Signal Scorecard**
| Signal | Rating | Reason |
|---|---|---|
| Owns the narrative / self-raises traps | Mixed | Self-raised idempotency & DLQ; missed PSP-retry ambiguity & webhook signing until pushed |
| Leads with trade-offs vs named alternatives | Mixed | Excellent on cursor-vs-offset; elsewhere stated choices without the rejected alternative |
| Pushes scale until it breaks | Mixed | Handled sharding & hot shard when pushed; never self-initiated the break |
| API as a designed contract | Strong | Explicit shapes, pagination + reasoning, idempotency, refund-as-resource, correct async code |
| Operability / second-order concerns | Weak | No reconciliation, no monitoring of stuck payments, no outbound-webhook reliability, no cost |
| Pace | Weak | 72 min; ~20 min lost to diagram-deferrals before answering |

**Overall level read:** Solid mid-level, knocking on senior for API & correctness, not yet a senior strong-hire. Real-loop verdict: Hire at mid-level / borderline-senior. Held below the bar by being led to the two hardest insights, near-absent operability, and 50% over the time budget.

**What a senior strong-hire would have done on THIS problem:**
- Self-raised the PSP idempotency/timeout-ambiguity trap in the HLD, not after two prompts.
- Designed outbound-webhook reliability unprompted: HMAC signature header, event ID for merchant dedupe, retry-with-backoff + outbound DLQ, poll endpoint as documented fallback.
- Built a reconciliation job comparing DB state vs PSP truth to resolve stuck `pending` payments — the operational backbone of any payment system.
- Modeled payment status as a state machine with allowed transitions to survive out-of-order PSP webhooks (webhook can arrive before the worker's own write).
- Owned monitoring: alert on age-in-pending, consumer lag, DLQ depth.
- Named the shard trade-off out loud (merchantId+paymentId spreads whale but makes list-by-merchant a scatter-gather → add a read model).
- Paced to finish HLD by ~25 min and spend the back half on PSP idempotency + reconciliation.

**Drill:** Review system_design_senior_guidance.md, focusing on signal #5 (operability) and #6 (pace) — the two Weak ratings. Force "how do I know it's broken?" and "what's the reconciliation/repair path?" for every design, unprompted.

### Diagram Quality
- Faithful to narration — every described box and flow present, nothing invented.
- Gaps mirror verbal gaps — no reconciliation, no outbound-webhook retry/DLQ, no monitoring.
- Process note: diagram used as a deferral mechanism rather than a thinking tool (three "update the drawio" deferrals).

**Time Taken: 72 minutes** (target 45–50; over by ~22).
