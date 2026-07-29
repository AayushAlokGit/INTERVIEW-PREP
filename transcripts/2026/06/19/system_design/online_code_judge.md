# System Design Round Transcript
**Date:** 2026-06-19
**Start Time:** 11:19
**End Time:** 12:20
**Duration:** 61 minutes
**Problem:** Design an Online Code Judge / Coding-Contest Platform (LeetCode / Codeforces style)

---

## Conversation Log

**Interviewer:** Introduced the problem (browse problems, submit code, sandboxed execution vs hidden tests, verdict, timed contests with live leaderboard). Asked for requirements with derived numbers.

**Aayush:** Asked if 10M DAU, 10 views/user/day, 5 submissions/user/day is fine.

**Interviewer:** Accept it but own it; derive submissions/sec steady + peak, contest spike, verdict latency, storage.

**Aayush:**
FRs: browse questions; code & submit in languages; execute vs testcases & record result; timed contests; live leaderboard.
NFRs: sandboxed/isolated/resource-restricted/secure execution; low-latency execution p99<200ms; HA 99.9%; eventual consistency for submissions OK; leaderboard <5s during contests; 10M DAU, 10 views/day, 5 subs/day; ~1k reads/s, 500 writes/s avg, peak 5x; ~10KB/submission, 500GB/day.

**Interviewer:** Challenged (1) p99<200ms for code execution — is that realistic, or conflating two latencies? (2) "peak 5x" ignores the contest thundering herd — quantify it.

**Aayush:** 200ms is API ack; execution is async (hence eventual consistency for submissions). Peak writes 50k–100k/s during contests.

**Interviewer:** Confirmed (~100–170x steady state). Rendered FRs/NFRs. Moved to core entities.

**Aayush:** Question(id,name,tags,description,testcases:[]); User(id,name); Submission(id,userId,questionId,result,logs,createdAt); Contest(id,name,questions:[],timeLimit,contestDate); Leaderboard(contestId,users:[]).

**Interviewer:** Probed: Submission missing code/language/status lifecycle + where the code blob lives; testcases embedded vs separate (size); is Leaderboard stored or derived, and is users:[] really one row for 100k participants?

**Aayush:** Submission needs status (submitted|executing|failed|accepted), code, language. TestCase a separate entity referencing question. Leaderboard is materialized.

**Interviewer:** Locked entities. Moved to API — verbs, paths, request/response shapes with explicit fields, pagination, idempotency, async verdict retrieval.

**Aayush:**
- Identity from JWT.
- GET /questions?cursor=&limit=&tags= -> [Question(id,name,tags)]
- GET /questions/:id?language= -> Question(id,name,tags,description), TestCases[], code stub
- POST /questions/:id/submissions {code,language} + Idempotency-Key -> 2xx Submission(id,..) scheduled
- GET /questions/:id/submissions/:id -> Submission(id,status,logs,result); client polls to terminal status
- POST /contests/:id/join -> 2xx
- GET /contests/:id/leaderboard?cursor=&limit= -> [User(id,name,score,rank)]

**Interviewer:** Flagged that returning hidden TestCases to client lets users game the judge — what do you actually return and where do hidden tests live?

**Aayush:** Only sample testcases to client; real testcases live in DB, fetched by the executor server-side before running.

**Interviewer:** Rendered API. Asked for full HLD: write path end-to-end + read paths, naming components, datastores, queues, flows.

**Aayush:** Client -> API Gateway (auth) -> API Service. API Service creates submission in Cassandra (write-optimized, AP; eventual consistency acceptable for submission status). API Service also enqueues a message to Kafka (durable, at-least-once; tradeoff: operational complexity). Idempotency key dedups in API service. Worker consumers pick up messages, spin up isolated/secure sandbox containers, fetch testcases from DB, run code under resource constraints. Partition Kafka by userId for per-user ordering. At contest peak, horizontally scale workers + containers. If submission is part of a contest and accepted, update user's score in a ContestScore entity. Read leaderboard: query Cassandra ContestScore partitioned by contestId, sorted by score, kept updated by write path. If read load too high, offload leaderboard to Redis (in-memory), reducing DB hot reads but introducing Redis<->DB consistency; Redis crash would hammer DB; keep Redis updated via CDC events from DB.

**Interviewer:** Rendered HLD faithfully. Deep Dive 1 (leaderboard): (a) 100k participants on one contestId partition — what breaks? (b) score is a mutable clustering key — how do you move a row when score changes; is it cheap?

**Aayush:** (a) Hot partition melts; split across multiple partitions to distribute load, at the cost of read-time latency (fan-out). (b) [pivoted] sort by userId then, score sort in memory; scatter-gather across partitions for top-N.

**Interviewer:** Pushed: keying by userId makes updates cheap but forces in-memory sort of 100k on every read — brutal. What structure gives BOTH O(log n) updates AND cheap Top-K? (You named it earlier.) And how to assemble global Top-100 from N shards?

**Aayush:** Redis sorted set. Pull top 100 from each shard, merge to get the best 100.

**Interviewer:** Correct (ZADD update O(log n), ZREVRANGE top-K; merge top-K from every shard). Noted global single-user rank across shards is the harder open problem. Deep Dive 2 (execution path): (a) at-least-once means a worker can run the same submission twice — what happens to ContestScore and how to make scoring idempotent? (b) 100k/s spike + container cold starts — what breaks, what happens to verdict latency/leaderboard freshness, how to design the fleet?

**Aayush:** (a) Make score update idempotent via atomic check-and-update of submission status. (b) Cold starts cause submissions to accumulate in Kafka -> backpressure; fix with serverless on-demand compute (e.g. Azure Durable Functions) or keep a pool of live containers and scale on CPU/memory.

**Interviewer:** Refined: CAS on status (only the worker that flips executing->accepted applies score) + storing absolute best per (user,problem) makes it idempotent. Corrected (b): submits still succeed (queue decouples); what degrades is verdict latency + leaderboard freshness = graceful degradation. Final question: is CPU the right autoscale signal, and how do you know the leaderboard breached its 5s SLA?

**Aayush:** Consumer lag is the primary metric for pile-up; scale execution containers on lag.

**Interviewer:** Confirmed consumer lag > CPU. Added the missing piece: a freshness metric (now - last_applied_score_event_ts) on the CDC->Redis leg to catch a stall there. Wrapped.

---

## Design Summary
**Requirements Gathered:** 5 FRs (browse, submit multi-language, sandboxed execution + verdict, timed contests, live leaderboard). NFRs: secure sandboxed execution; API ack p99<200ms with async execution; HA 99.9%; eventual consistency for submissions; leaderboard <5s freshness. Scale: 10M DAU, ~580 writes/s + ~1.1k reads/s avg, contest spike 50k–100k subs/s (~100–170x), 500 GB/day.

**High-Level Architecture:** Client -> API Gateway (JWT) -> API Service -> Cassandra (submissions, testcases, ContestScore) + Kafka (at-least-once, partition by userId) -> Worker consumers -> sandboxed containers (fetch hidden tests, run code, write status/result/score). Leaderboard read from Cassandra ContestScore (partition by contestId), optionally offloaded to Redis via CDC.

**Key Design Decisions & Trade-offs:** Cassandra for write-optimized AP store (eventual consistency justified vs submission-status requirement); Kafka at-least-once (durability vs op complexity); partition-by-userId for per-user ordering; idempotency key on submit; CDC->Redis for hot leaderboard reads; Redis ZSET for O(log n) updates + Top-K; scatter-gather Top-K across sharded ZSETs; CAS-on-status for idempotent scoring; consumer-lag-based autoscaling of execution fleet; graceful degradation (submits survive, verdicts lag).

**Scalability & Fault Tolerance Points:** Hot-partition fix via sharding; pre-warmed container pool + autoscale on consumer lag; idempotent execution/scoring under at-least-once; queue decoupling for backpressure absorption.

**Gaps / Missed Areas:** Clustering-key-mutation problem needed 3 nudges; Redis durability/source-of-truth model raised then dropped; leaderboard freshness metric had to be supplied; global per-user rank across shards left open; cost of container fleet and submission fairness/priority (contest vs practice) not discussed; over-ran time (61 min).

---

## Feedback Given

**Time Taken: 61 minutes**

### Requirements Clarification — 8.5/10
Clean FR/NFR split with derived numbers (50M/day -> ~580/s, 100M views -> ~1.1k/s, 500GB/day). Correctly separated API-ack latency from async execution latency under challenge, and named the contest spike (~100–170x) as sizing-driver. Minor: asked permission for scale assumption instead of asserting it.

### Core Entities — 8/10
Self-corrected Submission lifecycle (status/code/language), split TestCase out, recognized leaderboard as materialized. Needed a nudge that embedding testcases in Question is wrong.

### API Design — 9/10
Strongest phase, historically weakest area. Cursor pagination, idempotency key on the expensive write, JWT identity, async poll-for-verdict with terminal-status semantics, paginated leaderboard with concrete fields. Caught hidden-testcase exposure after a nudge. At senior bar.

### High-Level Architecture — 8.5/10
Coherent, trade-off-led: Cassandra (AP, justified), Kafka (at-least-once, partition-by-userId), worker+sandbox fleet, CDC->Redis. Named alternatives alongside choices.

### Deep Dive Quality — 7.5/10
Hot partition immediate; clustering-key-mutation took 3 nudges to reach the Redis ZSET answer he'd mentioned himself. Scatter-gather Top-K crisp once there. Deep-dive 2 stronger/faster: CAS-on-status idempotency, pre-warmed pool + autoscale, consumer-lag signal with less prompting.

### Scalability & Fault Tolerance — 8/10
Sharded ZSET, graceful degradation, idempotency under at-least-once. Gaps: Redis durability (source vs cache, rebuild) raised then dropped; freshness metric had to be supplied.

### Communication — 7.5/10
Clear, trade-off language throughout. Several key insights needed multiple prompts; dropped a sub-question again (answered only the consumer-lag half of the staleness question).

### Senior-Signal Scorecard
- Own the narrative / self-raise traps: **Mixed** — self-raised idempotency key, hidden-tests (after nudge), EC justification; clustering-key + freshness traps extracted.
- Lead with trade-offs vs named alternatives: **Strong** — consistent choice+sacrifice pairing.
- Push scale until it breaks: **Strong** — named the spike himself, engaged both breaks.
- API as designed contract: **Strong** — shapes, pagination, idempotency, security.
- Operability / second-order: **Mixed** — consumer-lag good; Redis durability dropped, freshness metric handed over, cost not discussed.
- Pace: **Weak** — 61 min vs 45–50; leaderboard deep-dive over-ran.

**Overall level read: Senior / hire (borderline strong-hire).** Architecture, trade-off discipline, API rigor at senior bar. Held back by pace and extraction-over-initiative on last-mile traps.

### What a senior strong-hire would have done on THIS problem
- Self-raised the clustering-key-mutation issue the moment "Cassandra sorted by score" was said -> "that's a Redis ZSET, not a Cassandra table."
- Named the Redis durability/consistency model up front (rebuildable cache, score events sourced in Cassandra, replay on loss).
- Owned the 5s freshness SLA with a concrete metric (now - last_applied_event_ts on CDC->Redis leg).
- Tackled global per-user rank across shards (score-histogram/bucketed approximation + local exact rank).
- Discussed container-fleet cost at 100k/s and submission fairness/priority (contest vs practice queue).
- Finished in ~50 min by collapsing the leaderboard deep-dive into one decisive move.

Pointed to system_design_senior_guidance.md items #1 (self-raise traps) and #6 (pace) to self-drill.
