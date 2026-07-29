# System Design Round Transcript
**Date:** 2026-04-29
**Start Time:** 16:56
**End Time:** 20:14
**Duration:** ~3h 18m wall clock (includes multiple pauses)
**Problem:** Design a Notification System

---

## Conversation Log

**Interviewer:** Please open the draw.io file at `C:\Users\aayus\Desktop\Interview Prep\transcripts\2026-04-29\system_design\notification_system.drawio`. Note the current time — we'll use it to track total round duration. Here's the problem: Design a Notification System similar to what companies like Uber, LinkedIn, or Rippling use to send notifications across multiple channels: push, email, SMS, and in-app alerts. Start by gathering requirements.

**Aayush:** [Wrote FRs in diagram] Functional Requirements: 1. Send notifications to users via various channels (email, push, SMS). 2. Consider user preference for notification channel. 3. Support scheduled notifications.

**Interviewer:** Good start. What about in-app notifications? Who sends notifications — internal services or external clients? Does the sender need to know if a notification was delivered/read? Also — give me NFRs with numbers: scale, latency, reliability?

**Aayush:** Updated FRs to include: in-app as a channel, internal services as senders, senders can track notification status.

**Interviewer:** Good FR set. Now NFRs — put numbers on scale, latency, reliability.

**Aayush:** [Paused the round]

**Aayush:** [Resumed] NFRs: 1. At-least-once delivery. 2. Max delivery delay of 1 min from scheduled time. 3. Notification service must have high availability to receive notifications from senders.

**Interviewer:** Two things missing: scale numbers (notifications/sec?) and availability SLA (99.9%? 99.99%?). What order of magnitude are we designing for?

**Aayush:** Asked for help with scale numbers.

**Interviewer:** [Provided derivation] Assume 10M users, 5 notifications/day/user = 50M/day ≈ 580/sec avg, ~5K/sec peak. 50GB data/day at 1KB/notification. State these as your assumption.

**Aayush:** Core entities: Notification, UserPreferences, User.

**Interviewer:** Good. What fields does each carry? What does UserPreferences store specifically?

**Aayush:** APIs: 1. POST /notifications → returns notification_id { channel, notificationTime, userId, content }. 2. GET /notifications/:id. 3. POST /users/:id/userPreferences { preferredChannels }.

**Interviewer:** Missing from POST: priority, idempotency key. GET response shape? POST should be PUT for preferences. What about bulk notifications?

**Aayush:** Updated: POST /notifications adds priority, idempotency via hash(notificationTime+userId+channel+priority). GET returns Partial<Notification>. PUT /users/:id/userPreferences.

**Interviewer:** Idempotency key concern: composite key can collide for two legitimate notifications. GET response should list explicit fields not TypeScript types. API design solid enough — move to high-level architecture.

**Aayush:** [Drew in diagram] Sender Services → API Gateway → Notification Service → SQL DB. Notification Service → SQS (with Email Channel, SMS Channel, In-App Channel) → per-channel workers. Added annotation: checks UserPreferences before queuing, creates DB record status=pending.

**Interviewer:** Good skeleton. Gaps: (1) Scheduled notifications — notificationTime in API but no scheduler. (2) Third-party providers for actual delivery. (3) Retry/status update — how do workers update DB? (4) In-app delivery mechanism — WebSocket? SSE? Polling? (5) No load balancer for Notification Service.

**Aayush:** [Updated diagram] Added API Gateway, cron job (minute interval) that reads DB and pushes to SQS, third-party email and SMS providers downstream of workers. Added SSE note for In-App workers.

**Interviewer:** Good additions. Probing: (1) What prevents cron from picking same notification twice? (2) In-App workers — no downstream component. How does SSE routing work across multiple instances? (3) Workers → DB status update edge missing. (4) Duplicate Notification Service box in diagram.

**Aayush:** Notification workers will ensure idempotency to prevent duplicate deliveries.

**Interviewer:** That handles worker-level deduplication but not cron-level. Cron can push same notification to SQS twice before any worker sees it. Need status transition: pending → queued atomically at cron time. In-app delivery mechanism?

**Aayush:** In-app delivery will use Server Sent Events.

**Interviewer:** Good choice. Connection management — with multiple Notification Service instances, how does In-App worker know which instance holds the user's SSE connection? What about offline users?

**Aayush:** [Stuck, asked for help completing the diagram]

**Interviewer:** [Explained solutions] (1) Cron: atomically UPDATE status=queued WHERE status=pending. (2) Redis Pub/Sub keyed by userId — In-App workers publish, Notification Service instances subscribe. (3) Offline: store status=pending_inapp, fetch on reconnect. (4) Workers write delivered/failed back to DB. (5) DLQs after max retries.

**Aayush:** Asked for diagram to be updated with missing components.

**Interviewer:** [Added to diagram] DLQs (red), Redis Pub/Sub (blue), User Clients SSE (green), status update arrows (dashed), offline fallback note.

**Aayush:** Asked for spacing/layout improvements.

**Interviewer:** [Rewrote diagram with clean 5-layer layout]

**Aayush:** Ready for feedback.

---

## Design Summary

**Requirements Gathered:**
- Functional: Multi-channel notifications (email, SMS, push, in-app), user preferences, scheduled notifications, internal senders, delivery status tracking
- Non-Functional: At-least-once delivery, max 1 min delivery delay, high availability for ingestion (99.99%)
- Scale: 10M DAU, 50M notifications/day, ~580/sec avg, ~5K/sec peak, 50GB/day

**High-Level Architecture:**
- Sender Services → API Gateway → Notification Service → SQL DB
- Notification Service checks UserPreferences → routes to SQS (per-channel queues: Email, SMS, In-App)
- Per-channel Workers consume queues → call 3P providers (SendGrid, Twilio) or Redis Pub/Sub for in-app
- Cron job (1 min) polls DB for pending scheduled notifications → atomically marks queued → pushes to SQS
- Redis Pub/Sub routes in-app notifications to correct Notification Service instance holding SSE connection
- DLQs on each channel for messages exhausting retries
- Workers write delivered/failed status back to SQL DB

**Key Design Decisions & Trade-offs:**
- SQL DB for notification storage (status tracking, scheduled query, ACID guarantees)
- SQS for async fan-out per channel (separate scaling, failure isolation)
- SSE for in-app delivery (unidirectional, lightweight)
- Redis Pub/Sub for SSE routing across instances
- Cron + DB status transition (pending→queued) for scheduled notification idempotency

**Scalability & Fault Tolerance Points:**
- At-least-once via SQS retry + DLQ
- Idempotency at worker level + cron-level status transition
- Per-channel worker pools scale independently
- Redis Pub/Sub for stateless Notification Service instances

**Gaps / Missed Areas:**
- No load balancer / horizontal scaling discussion for Notification Service
- No rate limiting per user
- No circuit breaker pattern for 3P provider failures
- NFR scale numbers required prompting (couldn't derive independently)
- Trade-off reasoning never stated for component choices
- API response shape for GET was vague ("Partial<Notification>")
- DLQs, Redis, SSE routing, status update arrows all required interviewer scaffolding

---

## Feedback Given

### 1. Requirements Clarification — 6/10
**Functional:** Strong — all 5 FRs identified, in-app added quickly.
**Non-Functional:** Weak — qualitative without numbers. Scale numbers had to be provided. NFRs should be quantified in first 3-4 minutes independently.

### 2. High-Level Architecture — 7/10
Solid skeleton. API Gateway → Notification Service → SQS → per-channel workers → 3P providers is the right shape. Cron job for scheduled notifications was a good independent decision.
Gaps: no load balancer discussion, no rate limiting, no circuit breakers.

### 3. API Design — 6/10
Started weak — missing content, priority, idempotency. Needed multiple rounds of prompting. PUT vs POST correction needed. GET response described as TypeScript type not explicit fields.

### 4. Component Design & Trade-offs — 5/10
Right choices but reasoning never stated. Should articulate: SQL vs NoSQL, SQS vs Kafka, SSE vs WebSockets.

### 5. Scalability & Fault Tolerance — 5/10
Covered with prompting: DLQs, cron idempotency, Redis for SSE routing.
Not covered: horizontal scaling, circuit breakers, rate limiting, DB outage behavior.

### 6. Deep Dive Quality — 5/10
Cron idempotency and SSE routing required heavy scaffolding. Status update flow needed repeated probing.
Practice pattern: for every component, ask — failure modes? state updates? race conditions?

### 7. Real-World Experience — 4/10
Relevant experience (Rippling payroll, async pipelines) not leveraged. Missed opportunity to draw parallels.

### 8. Communication — 6/10
Diagram helped organize thinking. Verbal explanations clear when answer was known. Required multiple prompts for full articulation. Asked for help completing own diagram — red flag in real interview.

### Diagram Quality — 6/10
**Strengths:** Clean layered layout, correct directional flow, per-channel queue separation, cron job behavior labeled, DB schema with indexes.
**Gaps:** DLQs, Redis, status arrows, SSE client all required interviewer completion. No load balancer drawn. In-app mechanism was a text note, not a drawn component.

### Summary Scores
| Category | Score |
|---|---|
| Requirements (FR + NFR) | 6/10 |
| High-Level Architecture | 7/10 |
| API Design | 6/10 |
| Component Design & Trade-offs | 5/10 |
| Scalability & Fault Tolerance | 5/10 |
| Deep Dive Quality | 5/10 |
| Real-World Experience | 4/10 |
| Communication | 6/10 |
| **Overall** | **5.5/10** |

### Top 3 Things to Fix
1. Derive NFR numbers yourself — DAU → requests/sec → storage/day. Do this in first 5 minutes.
2. State trade-offs explicitly — for every component choice say why you picked it over the alternative.
3. Complete your own deep dives — for every component: failure modes, state updates, race conditions. Don't wait to be asked.

**Time Taken: ~3h 18m wall clock (with pauses)**
