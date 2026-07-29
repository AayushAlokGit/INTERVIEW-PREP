# System Design Round Transcript
**Date:** 2026-05-30
**Start Time:** 8:22 PM
**End Time:** 9:56 PM
**Duration:** 94 minutes
**Problem:** Design a Distributed Job Scheduler

---

## Conversation Log

**Interviewer:** Hi Aayush — let's get started. Please note the current time. Your problem today: Design a Distributed Job Scheduler. Think of a system that lets users (or internal services) schedule jobs to run — one-off jobs at a specific time, or recurring jobs on a cron schedule. Jobs should run reliably across a fleet of worker machines. I've created a draw.io file for you. Begin with requirements gathering.

**Aayush:** 8:22

**Interviewer:** Start time noted (8:22 PM). Go ahead — start gathering requirements.

**Aayush:** FRs:
1. Support scheduling recurring and one time jobs.
2. Execute jobs on worker machines
3. Support status tracking of scheduled jobs
4. Support 3 retries for failed jobs
5. Support observability into each execution attempt of job

**Interviewer:** Clarifying probes — what is a "job" (code vs webhook vs registered task)? Who schedules (users vs services, multi-tenant)? Retries fixed or configurable? Cancellation/updates allowed? At-least-once vs exactly-once? Anything explicitly out of scope?

**Aayush:**
1. User-specified code we need to run
2. Internal services schedule
3. Retries configurable per job
4. Yes users can cancel/alter before run
5. At-least-once
6. Out of scope: workflow engine (durability, checkpointing, recovery)

**Interviewer:** How is "user-specified code" delivered — Docker image, uploaded artifact, registered code? Then give NFRs: scale (writes, dispatch/sec, total stored), latency SLA, availability, durability.

**Aayush:** Clients create docker images and workers pull them. Add FRs to drawio.

**Interviewer:** [Updated drawio with FRs.] Back to NFRs please.

**Aayush:** NFRs:
1. Low latency between scheduled time and execution time (<3s)
2. Durability for submitted jobs
3. 99.99% availability for submission; eventually consistent for status
4. 1k jobs submitted/s steady, 10k/s peak; 10k concurrent executions; 10k writes/s peak

**Interviewer:** Three probes — is 10k peak submission or dispatch? Total active scheduled jobs in storage? Retention of execution history?

**Aayush:** Dispatch rate for now; 10k/s peak submission for immediate one-offs. Each attempt stored 7 days then S3 cold. Each execution attempt for some time period.

**Interviewer:** That last answer was about attempts again — I asked about active scheduled jobs (pending future jobs sitting in DB).

**Aayush:** Active scheduled jobs and dispatch rate match right?

**Interviewer:** No — flow vs stock. Little's Law: active ≈ submission_rate × avg_lead_time. If 1k/s and 1h lead time, ~3.6M pending rows.

**Aayush:** Average lead time 1 hour, 3.6M pending rows.

**Interviewer:** Now HLD — components, storage, end-to-end flow.

**Aayush:** Core entities:
1. Job (id, dockerImageUrl, createdAt, chronScheduleId, scheduledAt)
2. JobExecutionAttempt (id, jobId, attemptNumber, logs, error, status, createdAt)
3. ChronSchedule (id, cron_details)

**Interviewer:** Entities noted. Gaps: no status on Job, no retry config, scheduledAt + cronScheduleId mixed. But you've described data model, not architecture — services, datastores, flow.

**Aayush:** Updated:
1. Job (id, dockerImageUrl, createdAt, scheduledAt, status: scheduled|running|cancelled, retryConfig)
2. JobExecutionAttempt (unchanged)
3. ChronSchedule (unchanged)
4. RecurringJob (id, dockerImageUrl, createdAt, chronScheduleId, retryConfig)
For recurring jobs, materialize one Job record per valid recurrence.

**Interviewer:** Cleaner. Now architecture please.

**Aayush:** Add core entities to drawio.

**Interviewer:** [Added entities.] API design first?

**Aayush:** API design:
- POST /jobs (idempotency key) → Job(id)
- POST /recurringJobs (idempotency key) → RecurringJob(id)
- GET /jobs/:id?isRecurring → Job/(s), attempts
- PUT /jobs/:id with {shouldCancel, changeSchedule, newChronScheduleId}

**Interviewer:** Response shapes vague. GET overloaded. PUT semantics awkward — use DELETE for cancel, PATCH for reschedule, separate attempts under /jobs/:id/attempts with pagination. Also missing list endpoint and recurring update semantics. Refine and then architecture please.

**Aayush:** [Refined APIs — 9 endpoints, separating GET attempts, DELETE for cancel with 409, PATCH for reschedule, separate recurring endpoints.] Add to drawio.

**Interviewer:** [Added APIs.] Architecture please — third ask. Components, storage, dispatch loop, workers, recurring materialization.

**Aayush:** Clients hit POST through load balancer to job service. Job service writes DB. For one-off jobs, pushes to SQS with visibility timeout = scheduledAt - now. Workers consume SQS, check if first attempt, create JobExecutionAttempt, execute. On error, push another SQS message respecting retry config. Recurring jobs handled by a separate polling process that materializes Job records and pushes to SQS.

**Interviewer:** Several issues:
1. SQS DelaySeconds max = 15 min, VisibilityTimeout max = 12h. Can't push days-out jobs.
2. Cancellation — can't delete specific SQS messages.
3. Retry duplication — original visibility timeout expiry + new push = duplicates.
4. First-attempt check has race.
5. Recurring poller single point of failure.
6. DB choice unspecified.
Pick two and address.

**Aayush:**
1. Scheduled dispatcher process polls DB and creates SQS messages, similar to recurring dispatcher.
2. Mark DB record as cancelled; worker checks before executing.
4. Only one message per job instance so two workers can't grab same message.

**Interviewer:** #4 is wrong. SQS is at-least-once. Visibility timeout expiry on slow job → SQS redelivers same message → two workers run it. How do you actually prevent that?

**Aayush:** Even with distributed locks, need TTL on the lock to prevent hung worker holding forever — wouldn't it be better to increase visibility timeout itself?

**Interviewer:** At-least-once is fundamental. Bumping timeout trades duplicate risk for crash-recovery latency, can't win both. Network partitions still cause it. You don't know user code runtime up-front. Right approach: accept duplicates, make execution safe — atomic DB lease (conditional UPDATE setting status='running', lease_owner, lease_expires) + fencing token on writes.

**Aayush:** Could use Redis distributed lock — each worker atomically creates Redis entry with TTL, renews periodically. New workers check lock first.

**Interviewer:** Same pattern, externalized. Two trade-offs: Redis vs DB-row lease (durability) and fencing tokens (stale workers after GC pause). Which DB choice?

**Aayush:** DB row lease — durable and no extra component.

**Interviewer:** Agreed. Now deep dive on scheduled dispatcher poller: cadence, query/index, sharding across pollers, cron storm of 200k jobs at 9:00:00.

**Aayush:**
1+2. Poll every 1s; index (scheduledAt, status).
3. Multiple poller instances querying same SQL with different LIMIT/OFFSET.

**Interviewer:** #3 is broken — LIMIT/OFFSET doesn't partition work; concurrent pollers see overlapping rows. Alternatives: hash partition on job_id, SELECT FOR UPDATE SKIP LOCKED, or atomic UPDATE claim. Pick one. And #4 — cron storm?

**Aayush:** Hash partition on job_id — each poller selects one partition. Avoids double dispatch and removes locking latency.

**Interviewer:** Follow-ups — partition assignment (ZK/Consul)? Avoiding re-dispatch within a partition? Cron storm?

**Aayush:** ZooKeeper for partition-poller assignment and failure handover. Update status of job row to avoid re-dispatch. Cron storm needs many pollers, SQS scaling, larger worker pool.

**Interviewer:** Hand-wavy. Concrete numbers: 200k jobs at 9:00:00.000 must dispatch within 3s. 10k worker cap means storm can't actually clear in 3s. Real fix: jitter to smear the storm. Does your design meet SLA on this?

**Aayush:** Design does not meet 3s SLA on 200k storm with 10k workers. Better option: add slight jitter to scheduledAt for cron job instances; use priority levels to control jitter window.

**Interviewer:** Exactly right — used by Airflow, K8s cronjobs. Share current time.

**Aayush:** 9:56.

**Interviewer:** [Delivered feedback below.]

---

## Design Summary

**Requirements Gathered:**
- FRs: schedule recurring + one-off jobs (Docker images), execute on workers, status tracking, configurable per-job retries, per-attempt observability, cancel/alter before run, at-least-once.
- Out of scope: workflow engine (DAGs, checkpointing, recovery).
- NFRs: <3s dispatch latency, durability, 99.99% submission availability, eventually consistent status, 1k/s steady submission (10k/s peak), 10k/s dispatch peak, 10k concurrent executions, 3.6M pending jobs, 7-day hot retention for attempts then S3 cold.

**High-Level Architecture:**
- Internal services → LB → Job Service → DB.
- Hash-partitioned Scheduled Dispatcher Pollers (ZK-coordinated, polling every 1s on indexed `scheduledAt, status`) → SQS.
- Separate Recurring Job Dispatcher materializing Job rows from ChronSchedule → SQS.
- Worker Pool consuming SQS → pulls Docker image → atomic DB-row lease with TTL renewal → writes JobExecutionAttempt → reports status back.

**Key Design Decisions & Trade-offs:**
- Materialize recurring instances as Job rows for uniform execution path.
- Idempotency keys on POSTs; PATCH/DELETE separation for update/cancel; 409 on cancel-after-run.
- DB-row lease (durability) chosen over Redis lock (less durable, extra component).
- Hash partition on job_id (no lock contention) over LIMIT/OFFSET (broken) or FOR UPDATE SKIP LOCKED.
- Cancellation via row flag + worker check before execute.
- Jitter + priority for cron storm.

**Scalability & Fault Tolerance Points:**
- Hash-partitioned pollers with ZK handover on failure.
- DB-row lease with TTL + renewal for at-least-once safety.
- Index `(scheduledAt, status)` for due-jobs query on 3.6M rows.
- Jitter-by-priority for thundering-herd cron storms.

**Gaps / Missed Areas:**
- Never named the database concretely.
- Fencing tokens not raised until prompted.
- No DLQ / poison pill handling.
- No worker isolation/security for untrusted user Docker code.
- No multi-region or DB failover discussion.
- No response-shape detail or pagination on API.
- No list endpoint, no rate limits.
- Cancellation race (worker reads "not cancelled" just before user cancels) not discussed.
- Recurring update/pause API missing.
- Diagram lagged verbal design; user drew nothing himself.

---

## Feedback Given

**Time Taken: 94 minutes** (target 45–60, significant overrun)

### Requirements Clarification — 7/10
Strong: clear FRs, explicit out-of-scope, concrete NFR numbers, committed to at-least-once.
Weak: didn't volunteer NFRs; confused dispatch rate with active stock (Little's Law gap); retention question answered for wrong thing; no execution-path availability target; no multi-tenant rate limit mention.

### High-Level Architecture — 6/10
Strong: eventually clean architecture; separated recurring vs one-off dispatchers; hash partitioning idea is right.
Weak: HLD required three prompts; initial SQS-visibility-timeout-as-delay was a fundamental error showing queue-fundamentals gap; no LB or cache mentioned until prompted; DB never named.

### API Design — 7.5/10
Strong: idempotency keys (improvement from prior sessions), JWT auth, clean revision with paginated attempts and 409 semantics.
Weak: response shapes still under-specified; no pagination cursor; no list endpoint; no rate-limit/quota mention.

### Component Design & Trade-offs — 6/10
Strong: DB lease over Redis with sound reasoning; hash partition correctly chosen.
Weak: almost no proactive trade-off reasoning; didn't engage with fencing tokens until prompted; small conceptual gap on visibility-timeout-as-lock.

### Scalability & Fault Tolerance — 6/10
Strong: honest cron-storm answer + correct jitter+priority mitigation; ZK for partition coordination.
Weak: hand-wavy at first ("more pollers, scale SQS"); no DLQ, no multi-region, no worker-isolation/security for untrusted Docker code.

### Deep Dive Quality — 5.5/10
Strong: hash partitions, DB-row lease with renewal, jitter eventually correct.
Weak: did not volunteer break/fix on own design; multiple wrong-then-corrected answers (LIMIT/OFFSET sharding; visibility timeout as fix for at-least-once).

### Communication & Process — 5/10
Required prompts for nearly every transition; mixed data model with architecture; 94 min over budget — chronic time-discipline issue.

### Diagram Quality — 6/10
User drew nothing himself; diagram constructed by interviewer at user's request. Diagram lagged verbal design throughout.
