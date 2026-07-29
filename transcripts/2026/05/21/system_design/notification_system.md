# System Design Round Transcript
**Date:** 2026-05-21
**Start Time:** 12:40
**End Time:** 14:05
**Duration:** 85 minutes
**Problem:** Design a Notification System

---

## Conversation Log

**Interviewer:** Presented the problem — design a notification system supporting push, SMS, and email across transactional, critical, and marketing categories. Asked Aayush to start with requirements gathering.

**Aayush:** Do we need to consider user preferences?

**Interviewer:** Yes — users opt in/out per channel and per category. Marketing must respect opt-outs; critical (OTP, security) bypasses preferences. Preferences managed elsewhere, exposed via API/store.

**Aayush:** What categories of notification do we need to support?

**Interviewer:** Transactional (medium priority), Critical (highest, latency-sensitive, bypass prefs), Marketing (low priority, high volume, respect opt-outs). Asked Aayush to commit to FRs and NFRs with numbers.

**Aayush:** Listed FRs: (1) multi-channel support (Push/SMS/Email), (2) track notification status, (3) categories with different priority, (4) user preferences for channel and category.

**Interviewer:** Noted scheduling/batch as missing; asked again for NFRs with arithmetic.

**Aayush:** Asked: do we need scheduled notifications and batch support?

**Interviewer:** Yes to both. Held Aayush to commit NFRs with numbers — assume 50M DAU.

**Aayush:** NFRs: (1) 99.99% availability for notification acceptance (~57 min downtime/year); (2) critical <500ms delivery delta, lower priority 5s; (3) 10 notifs/user/day → 500M/day → ~5k/s avg, ~50k/s peak (10x); (4) at-least-once delivery.

**Interviewer:** Corrected SLA math — 99.99% is ~52.6 min, not 57. Praised the acceptance-vs-delivery distinction. Asked for the API and high-level architecture.

**Aayush:** Core entities: Notification, User, UserPreferences. APIs: `POST /notifications` → 200 OK (creates notification); `GET /notifications/:id` → status/channel/category/recipients; `PUT /user/:id/preferences`. UserId from JWT.

**Interviewer:** Pressure-tested the write endpoint — status code & response body for a 5M-recipient batch, idempotency on retry, and batch status representation.

**Aayush:** Revised: `POST /notifications` returns Notification id, added `idempotency-key` header; `GET /notifications/:id` returns `RecipientStatus[]`.

**Interviewer:** Asked to confirm async model (202 Accepted) and pagination for the 5M-recipient status array.

**Aayush:** Agreed — 202 Accepted, add pagination.

**Interviewer:** Asked for high-level architecture drawn in draw.io.

**Aayush:** Updated the diagram and asked the interviewer to read it.

**Interviewer:** Read the diagram — three priority queues (Critical/Transactional/Batch), batch notif processor, scheduled-notification poller, per-channel workers, DLQs, API Gateway, 3P providers. Asked Aayush to walk through verbally and answer: where preferences are checked, how the notification service & DB scale, and how batch fan-out works.

**Aayush:** (1) Preferences checked just before pushing to the queue — in the notification service for critical/transactional queues, in the batch processor for the batch queue. (2) Horizontally scale notification service; move from SQL to a write-optimized store like DynamoDB (multi-master, quorum consensus, eventual consistency) since the workload is write-heavy and read load is light. (3) Batch processor writes 5M queue messages and 5M DB rows; iterating 5M users will take time.

**Interviewer:** Pushed on three threads — parallelizing the batch fan-out (and latency), the data-model contradiction (nested `recipientStatus` map vs 5M separate rows under concurrent worker updates), and the "read load is not much" claim (5M preference reads per batch).

**Aayush:** Added a `BatchMemberNotification` entity to track per-recipient status (separate rows); use a Redis cache for UserPreferences to avoid DB on the hot path; shard the 5M users across multiple workers to parallelize fan-out.

**Interviewer:** Probed two more — cache staleness (user opts out, then a campaign reads a stale "opted-in" cache → compliance violation) and duplicate delivery under at-least-once (worker sends OTP, crashes before ack).

**Aayush:** (1) Use CDC events from the DB on UserPreferences updates to refresh the Redis cache. (2) Save notification status to DB before acking the queue for an own idempotency mechanism; better, pass the unique notification ID as an idempotency key to the 3rd-party provider if supported.

**Interviewer:** Wrapped up. Collected timing and delivered feedback.

---

## Design Summary

**Requirements Gathered:**
- FRs: multi-channel (push/SMS/email); track notification status; prioritized categories (transactional/critical/marketing); user preferences per channel & category; scheduled notifications; batch/marketing sends.
- NFRs: 99.99% availability for notification acceptance; critical <500ms delivery delta, lower priority 5s; 50M DAU, ~500M notifs/day, ~5.8k/s avg, ~50k/s peak; at-least-once delivery.

**High-Level Architecture:**
- Client services → API Gateway (auth) → Notification Service.
- Notification Service writes to DB and routes to three priority queues: Critical, Transactional, Batch/Marketing.
- Batch notif processor fans out 5M-recipient campaigns into per-recipient queue messages + rows (sharded across workers).
- Background poller scans DB for notifications near scheduledTime and enqueues them.
- Per-channel workers consume queues, call 3rd-party SMS/email/push providers, and write status back.
- DLQs per queue for failed messages.

**Key Design Decisions & Trade-offs:**
- SQL → DynamoDB-style write-optimized store, accepting eventual consistency for write throughput.
- `BatchMemberNotification` table for per-recipient status to avoid hot-row contention.
- Redis cache for UserPreferences on the hot path, invalidated via CDC.
- 3rd-party idempotency key to handle duplicate delivery under at-least-once.

**Scalability & Fault Tolerance Points:**
- Horizontal scaling of the notification service; sharded batch workers; DLQs; CDC-driven cache invalidation; provider-side idempotency.

**Gaps / Missed Areas:**
- No load balancer named; rate limiting deferred ("future"); no circuit breaker / provider failover for 3rd-party providers.
- Diagram not synced with verbal updates (BatchMemberNotification, Redis cache, sharded workers not drawn).
- No explicit latency budget validation for the 500ms critical path.
- Idempotency-key storage/check location not specified.

---

## Feedback Given

# Feedback — Design a Notification System

## What went well
- Strong architectural skeleton: three priority queues, dedicated batch fan-out processor, per-channel workers, DLQs, scheduled-notification poller, 3rd-party provider abstraction.
- Stated reasoning for a trade-off (SQL → DynamoDB because write-heavy + tolerable eventual consistency).
- Good deep-dive answers once engaged: BatchMemberNotification for hot-row contention, Redis for preference reads, CDC for cache invalidation, 3rd-party idempotency key for duplicate delivery (correctly spotted the crash window in "write status before ack").
- Auth included in the API from the start.

## What to work on
1. Deferred committing to NFRs — asked four clarifying questions before any non-functional number; needed explicit holding. Get FRs + NFRs-with-numbers on the board fast, then refine.
2. SLA arithmetic slip — 99.99% is ~52.6 min/year, not 57. Memorize: 99.9% → 8.7 hrs, 99.99% → 52.6 min, 99.999% → 5.3 min.
3. API needed prompting on every refinement — idempotency key, 202 status, pagination, response body. Bake these in as defaults.
4. No load balancer named; rate limiting punted to "future"; no circuit breaker / failover on 3rd-party providers — the most likely incident source for this system.
5. Waits for the interviewer to point out the break. Drive the naive → break → fix → trade-off loop yourself.

## Diagram Quality
- Core components present (gateway, queues, DB, workers, DLQs, 3P providers). Missing: load balancer, Redis cache, explicit preference-check component.
- Verbal–diagram gap: BatchMemberNotification, Redis cache, and sharded batch workers were described but not drawn; entity box still shows the old nested-map recipientStatus.
- Cleanliness: overlapping duplicate worker boxes, a vague ellipse used as a stand-in for status updates, one edge mis-wired into a DLQ box instead of the Marketing queue.

## Scoring
| Criterion | Score |
|---|---|
| Requirements Clarification | 6.5 / 10 |
| High-Level Architecture | 7 / 10 |
| API Design | 6 / 10 |
| Component Design & Trade-offs | 7 / 10 |
| Scalability & Fault Tolerance | 6.5 / 10 |
| Deep Dive Quality | 6.5 / 10 |
| Communication | 6 / 10 |

**Overall: 6.5 / 10** — A competent mid-level performance with a solid design core. Architecture instincts are there; process is what holds the score down: commit to numbers faster, bake API best-practices in by default, drive the break/fix loop, and keep the diagram synced with the verbal design.

**Time Taken: 85 minutes**
