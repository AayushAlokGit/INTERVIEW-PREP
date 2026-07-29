# System Design Round Transcript
**Date:** 2026-04-30
**Start Time:** 9:39 AM
**End Time:** 11:09 AM
**Duration:** 90 minutes
**Problem:** Design a Distributed Rate Limiter

---

## Conversation Log

**Interviewer:** Note the current time and share it with me — that's our round start time. Here's your problem: Design a Distributed Rate Limiter. A large API platform wants to prevent abuse and ensure fair usage across its services. You need to design a rate limiting system that can enforce request quotas per user/API key, across multiple backend servers, at scale. Take a few minutes to gather requirements — ask me any clarifying questions you have, and establish the key functional and non-functional requirements before jumping into the design.

**Aayush:** 9:39. One user can have multiple API keys and the rate limiting must be enforced on a per API key level?

**Interviewer:** Yes, rate limiting is enforced per API key — one user can have multiple keys, and each key has its own independent quota. What else do you want to clarify?

**Aayush:** What would be the scale of API requests for rate limiting?

**Interviewer:** ~500 million API requests per day across all clients. Traffic is not uniform — expect peak bursts of up to 5x the average rate. What does that imply for your design? And what other requirements do you want to nail down?

**Aayush:** Can the rate limiting quota be configurable?

**Interviewer:** Yes — quotas are configurable per API key. Different keys can have different limits (e.g., free tier vs. paid tier). Keep going — what else?

**Aayush:** Return 429 requests when quota exhausted?

**Interviewer:** Yes — return HTTP 429 Too Many Requests when the quota is exceeded. You can also assume we should include a Retry-After header. You've confirmed a few functional requirements. Have you thought about the non-functional requirements?

**Aayush:** FRs -> 1. Rate limiting per API key/per user (one user can have multiple API keys) 2. Configurable rate limit quotas per API key 3. quota exhausted - 429 code

**Interviewer:** Good, those 3 FRs are solid. Now what about non-functional requirements? The system is handling 500M requests/day — what properties does this system need to have? Try to put numbers on them.

**Aayush:** NFRs -> 1. Highly available system 2. Low latency rate limiting (<5ms) 3. Scale to support 5k API requests/s avg and 25k API requests/s peak

**Interviewer:** Good — latency and scale numbers are solid. But "highly available" is vague. What availability SLA would you target? And what's the implication if the rate limiter goes down — do we fail open or fail closed? Also, what happens to accuracy when requests for the same API key hit different servers?

**Aayush:** What does fail open or fail closed mean?

**Interviewer:** Fail open: if the rate limiter is down, allow the request through anyway. Prioritizes availability over protection. Fail closed: if the rate limiter is down, reject/block the request. Prioritizes protection over availability. Which do you think is more appropriate here, and why?

**Aayush:** I believe failing closed would be better here to ensure that in downtime of rate limiter the API platform can't be flooded with requests.

**Interviewer:** That's a reasonable position — but think about the trade-off from the other side too. If your rate limiter fails closed and goes down, every legitimate request gets rejected until it recovers. For a platform where clients are paying for API access, that's a significant business impact. Most production systems actually fail open with an alert. What does that tell you about the availability requirement for your rate limiter?

**Aayush:** 99.99% availability, i.e. maximum availability to reduce abuse windows.

**Interviewer:** Good — 99.99% availability (roughly 52 minutes downtime/year). Lock that in. And on accuracy: in a distributed system, if two servers are both counting requests for the same API key independently, you might allow slightly more requests than the limit. Is that acceptable, or do you need exact enforcement?

**Aayush:** Given that the rate limiter is distributed both the servers would reach out to the common rate limiter interface so the numbers would be the same for both the servers right?

**Interviewer:** That's the right instinct — you're describing a centralized counter store that all servers talk to. So yes, you'd get consistent counts. But that centralized store is now on the hot path of every single API request. What does that mean for your latency and availability requirements on that store? And what happens if it goes down?

**Aayush:** Latency should be <5ms and availability should be 99.99%. NFRs -> 1. Highly available system (99.99%) - When rate limited down then allow the request (Allowing abuse during downtime windows) 2. Low latency rate limiting (<5ms) 3. Scale to support 5k API requests/s avg and 25k API requests/s peak

**Interviewer:** Good — and you just switched from fail closed to fail open after thinking through the trade-off. That's exactly the right process. NFR #4 is blank — what about accuracy/consistency?

**Aayush:** Exact enforcement would be desired.

**Interviewer:** Good. Lock that in as NFR #4: Exact rate limit enforcement. Now let's move to the high-level design. Walk me through your architecture.

**Aayush:** CoreEntities -> 1. APIKeyRateLimitConfiguration rateLimitQuota rateLimitingAlgorithm

**Interviewer:** Good start on the data model. Keep going — what fields does that entity have? And beyond data entities, walk me through the system components.

**Aayush:** APIs -> 1. PUT /APIKeyRateLimitConfiguration { apiKeyType, rateLimitQuota, rateLimitAlgorithm }

**Interviewer:** Good — that covers the config update API. A few quick probes: Who calls this endpoint? Shouldn't the API key ID be in the path? What does the response look like? What does apiKeyType represent?

**Aayush:** APIs -> 1. PUT /APIKeyRateLimitConfiguration -> Admin facing API. Returns 2xx { apiKeyType // Used to identify the type of API Key, rateLimitQuota, rateLimitAlgorithm }

**Interviewer:** Better — but on the REST convention point: if you're updating the config for a specific API key, shouldn't the identifier be in the path? And what's the other critical API here?

**Aayush:** apiKeyType would be like free/paid — we can't have an id for all API Keys and the config would be defined for particular API key types.

**Interviewer:** Interesting — so you're rate limiting at the tier level, not per individual API key. That contradicts FR #1 which said per API key with configurable quotas per key. Which model do you actually want?

**Aayush:** Rate limiting for an API key type like free/paid makes the system simpler for now.

**Interviewer:** Fair — but note this means all free-tier keys share the same quota config. A paying customer can't get a custom quota. Flag that as a known limitation. Now — what's the second critical API?

**Aayush:** CoreEntities -> 1. APIKeyRateLimitConfiguration apiKeyType rateLimitQuota rateLimitingAlgorithm 2. APIKeyType. APIs -> 1. PUT /api-keys/:api-key-type/rate-limit-config -> Admin facing API. Returns 2xx { rateLimitQuota, rateLimitAlgorithm } 2. PUT /rate-limit -> Return 429 or forwards request { request body }

**Interviewer:** Good progress. On API #2: HTTP verb is wrong (this isn't updating a resource), what goes in the request body, and what does a successful response look like?

**Aayush:** 2. GET /rate-limit?apiKeyType={} -> Return 429 or forwards request to required API and return APIs response

**Interviewer:** Good — GET makes more sense for a check operation. But the rate limiter shouldn't forward the request — that conflates responsibilities. Also, GET has a side-effect here (incrementing a counter). Does that change anything?

**Aayush:** What would you suggest?

**Interviewer:** I won't give you the answer — but think about it: what does the caller need to know when allowed? And GET semantically means read without side effects, but checking rate limit increments a counter. Which verb would you go with and why?

**Aayush:** CoreEntities -> [updated entities and APIs]. 2. POST /rate-limit -> Return 429 and when the quota resets, if rate limited, or returns remaining quota { apiKeyType }

**Interviewer:** Good — POST is the right call. One small thing: the 200 response body — what fields does it return?

**Aayush:** CoreEntities -> [updated]. 2. POST /rate-limit -> Return 429 and when the quota resets, if rate limited, or returns remaining quota { apiKeyType }

**Interviewer:** Good — that's a solid API definition. Now let's move to the high-level architecture. Walk me through the components.

**Aayush:** Client will make API request to API gateway and the API gateway will call the rate limiting endpoint. If rate limiter allows then forward request to backend services. Else return the 429 to client with details of when quota refreshes.

**Interviewer:** Good — that's the right flow. Now let's add detail. What's inside the Rate Limiter Service? What technology for counter storage and config storage?

**Aayush:** [Refers to diagram — drew Client → API Gateway → Rate Limiter Service → Redis with two key patterns]

**Interviewer:** Good diagram — I can see the flow. Redis key for the counter uses {userId}{apiKeyType} but earlier you said rate limiting is per apiKeyType, not per user. Which is it?

**Aayush:** The rate limiting would be per API keys/user but the configuration would be the same for a particular API key type.

**Interviewer:** Good clarification — config is per apiKeyType, counter is per userId+apiKeyType. Now: TTL on the counter and atomic increment — how do you handle those?

**Aayush:** TTL on counter depends on the rateLimitingConfig. Counter would be incremented using atomic Lua scripts.

**Interviewer:** Good on both counts. What does that Lua script actually do? Walk me through the logic.

**Aayush:** In each incoming request it would fetch the quota for a particular API key type, check the current request counter and then update the request counter.

**Interviewer:** Good — that's the right sequence. Let me sharpen it: Fetch quota from where? What's the check condition explicitly? What happens if the counter key doesn't exist yet?

**Aayush:** 1. Fetch quota from the config key in Redis 2. Check if counter exists for user if not then create the counter with TTL extracted from the config 3. Check if current counter >= quotaLimit if greater than quota limit then fail, return -1 or flag which rate limiter service will recognize and return 429 error 4. If counter <=quotaLimit then update the counter and return the remaining quota.

**Interviewer:** Solid Lua script flow. Now you have a single Redis instance — you need 99.99% availability at 25k RPS. What happens if it goes down or becomes a bottleneck?

**Aayush:** We can have a Redis cluster to increase redundancy and improve availability — master-slave type where all writes go to the master Redis instance and are async replicated to other nodes. In case of failure of master, one of the slaves is promoted to master.

**Interviewer:** Good — but you said async replication. If master fails and a slave gets promoted, what happens to counter increments not yet replicated? Given your NFR of exact enforcement — is that acceptable?

**Aayush:** Given that NFR states exact counting of requests the replication needs to be sync.

**Interviewer:** Good catch — you self-corrected. But sync replication adds latency. You have a <5ms latency NFR. How do you balance these two competing requirements?

**Aayush:** Sacrifice the exact counting for latency seems a good option at this point in the design.

**Interviewer:** Valid trade-off — but say it explicitly as a trade-off, not just a default. "We're relaxing exact enforcement in favor of <5ms latency. During a failover window, we may allow slightly more requests than the quota due to replication lag. We accept this as a known gap." Now — what about the Rate Limiter Service itself at 25k RPS?

**Aayush:** The rate limiter service is stateless and can be horizontally scaled to handle peak loads. We can scale based on monitoring memory and CPU consumption.

**Interviewer:** Good — stateless services are easy to scale horizontally. Who routes traffic to multiple instances? And for a rate limiter specifically, what metric is actually the better signal to scale on?

**Aayush:** CPU tracking would be recommended since rate limiter would have mostly CPU bound tasks.

**Interviewer:** Reasonable, but the Rate Limiter Service is mostly doing Redis reads/writes and running a Lua script — that's I/O bound, not CPU bound. The better signal is request rate (RPS) or Redis connection pool saturation. Now — you have rateLimitingAlgorithm in your config entity. What algorithm are you actually implementing?

**Aayush:** Implementing token bucket algorithm would be good as this would allow to handle the peak 5x burst traffic.

**Interviewer:** Good choice and good reasoning. Walk me through how token bucket works mechanically, and how you'd implement it in your Redis Lua script.

**Aayush:** I am blanking out on this.

**Interviewer:** [Gave hint: token bucket tracks current tokens and timestamp of last request. On each request: refill tokens based on time passed, then consume one token if available.]

**Aayush:** Token bucket tokens are refilled after specific time limits right, not on each request?

**Interviewer:** That's actually a common misconception — there are two variants: Periodic refill (bulk reset at intervals) vs Continuous refill (gradual based on elapsed time). Continuous is what most production rate limiters use. Given your 25k peak RPS, which would you pick?

**Aayush:** The continuous variant is good for the use case to prevent huge volume of requests at window edges.

**Interviewer:** Exactly right. The Lua script would: read currentTokens and lastRefillTimestamp, compute elapsed time, compute tokensToAdd capped at maxTokens, allow/deny, update. Can you think of any edge case in this logic at scale?

**Aayush:** No.

**Interviewer:** [Pointed out concurrent requests race condition — two servers reading currentTokens=1 simultaneously and both allowing.] How does your Lua script handle that?

**Aayush:** Not sure of that.

**Interviewer:** This is why you chose Lua scripts — Redis is single-threaded, so Lua scripts execute atomically. No two scripts can interleave on the same Redis instance. The atomicity guarantee comes from Redis, not your application code. Now — what happens when the Rate Limiter Service can't reach Redis? You said fail open — walk me through how you'd implement that.

**Aayush:** The rate limiter service would return some fixed code and allow the request to pass through.

**Interviewer:** Good — that's the right behavior for fail open. How does it detect Redis failure? What's the timeout value? Should these events be logged?

**Aayush:** A timeout of more than 5ms for a Redis request or a connection failure would be the cases when the rate limiter service would have to fail open. It could also log the issue observed causing the service to fail open for observability.

**Interviewer:** Good — timeout threshold tied to your latency NFR is smart. One more: if Redis is down and you're failing open, how would you know when Redis recovers so you can stop failing open?

**Aayush:** Not sure.

**Interviewer:** [Introduced circuit breaker pattern: closed state (normal), open state (fail open without trying Redis), half-open state (probe requests to check recovery).] Note the current time and we'll go into feedback.

**Aayush:** 11:09 AM.

---

## Design Summary

**Requirements Gathered:**
- FRs: Rate limiting per API key type / user (config per tier, counter per user), configurable quotas per API key type, 429 on quota exhaustion
- Out of scope: per-individual-API-key rate limiting, custom quota per user
- NFRs: 99.99% availability (fail open), <5ms latency, 5k avg / 25k peak RPS, exact enforcement (traded off for latency via async replication)

**High-Level Architecture:**
- Client → API Gateway → Load Balancer → Rate Limiter Service (horizontally scaled, stateless) → Redis master-slave cluster
- API Gateway → Backend Services (after rate limit check passes)
- Redis stores: config key (apiKeyType → RateLimitConfig) and counter key ({userId}{apiKeyType} → requestCounter with TTL)
- Atomic Lua script for check-and-increment

**Key Design Decisions & Trade-offs:**
- Rate limiting per API key type (tier) not per individual key — simplicity trade-off, noted as limitation
- Async replication in Redis cluster — exact counting traded off for <5ms latency budget
- Token bucket algorithm — chosen to handle 5x burst traffic
- Continuous refill variant — to prevent thundering herd at window reset
- Fail open on Redis failure — availability over protection, with logging

**Scalability & Fault Tolerance Points:**
- Rate Limiter Service is stateless → horizontal scaling behind load balancer
- Redis master-slave cluster for HA
- Fail open on Redis timeout (>5ms) or connection failure with observability logging

**Gaps / Missed Areas:**
- Circuit breaker pattern not self-initiated (introduced by interviewer)
- Token bucket mechanics blanked — needed hint to reconstruct
- Load balancer not in initial design — added after prompting
- Real-world experience (Azure, Rippling billing/telemetry) never leveraged
- CPU-based autoscaling suggested instead of RPS/connection pool
- Race condition in concurrent Lua access not self-identified

---

## Feedback Given

**Time Taken: 90 minutes**

### 1. Requirements Clarification — 3/5

FRs were solid and well-scoped. Correctly identified the per-tier vs per-key distinction and documented it as an explicit out-of-scope decision — that's mature design thinking.

NFRs needed significant prompting:
- "Highly available" without a number is not an NFR — needed two prompts to land on 99.99%
- Latency (<5ms) was self-initiated — good
- Scale numbers were given, but correctly derived avg/peak from them
- Accuracy/exact enforcement wasn't raised until prompted — this is a core property of a rate limiter

Pattern to fix: Before moving to design, ask yourself: "What are the hard constraints on this system?" — latency, availability SLA, accuracy, and consistency should be reflexive.

### 2. High-Level Architecture — 3.5/5

Clean flow: Client → API Gateway → Load Balancer → Rate Limiter Service → Redis. Separation of concerns is right.

Good: articulated the Redis master-slave trade-off (async replication = undercounting risk) clearly and documented it in the diagram.

Gap: Load balancer wasn't in the initial design — added only after prompting. For a stateless service at 25k RPS, this should be reflexive.

### 3. API Design — 2.5/5

Config API (PUT /api-keys/:api-key-type/rate-limit-config) — solid, correct REST conventions after correction.

Rate limit check API — needed three iterations: started as PUT (wrong), corrected to GET (better but side-effect concern), landed on POST (correct, with good reasoning).

Response shapes needed prompting in both cases. Final shape was clean once defined.

Pattern to fix: Before finalizing any API, explicitly state: verb, path, request body, success response shape, error response shape. Don't leave response bodies vague.

### 4. Component Design & Trade-offs — 3.5/5

Redis choice was correct and well-reasoned. Lua scripts for atomicity was a strong answer — correctly tied it to Redis single-threading as the atomicity guarantee.

Token bucket: correct algorithm choice with good reasoning (handles 5x burst). However, blanked on the mechanics and needed a hint to reconstruct the logic. Continuous vs periodic refill distinction understood once prompted.

Circuit breaker: not self-initiated. This is a standard resilience pattern for any service on the hot path — at this experience level it should be in the toolkit.

Pattern to fix: When you choose a technology or algorithm, immediately explain why — "I'm using Redis because it's in-memory, single-threaded, and supports atomic Lua scripts which gives us both speed and consistency" should come unprompted.

### 5. Scalability & Fault Tolerance — 3/5

Horizontal scaling of Rate Limiter Service: correctly identified as stateless — good.

Redis HA: master-slave with async/sync trade-off articulated well — good.

Fail open: initially said fail closed, reversed after thinking through business impact — right process, but shouldn't need prompting to think through both sides.

Circuit breaker: unknown. Directly applicable to Redis failure handling — closed/open/half-open states prevent thundering herd on recovery.

### 6. Deep Dive Quality — 3/5

Lua script logic: solid once prompted, but token bucket mechanics needed a hint. The race condition question was answered correctly once pointed at.

The async replication gap (undercounting during failover) was self-corrected after prompting — good recovery.

Redis failure handling: partial. Got timeout detection and logging, but needed circuit breaker pattern introduced.

### 7. Real-World Experience — 1.5/5

Never drew on Microsoft or Rippling experience unprompted — not once. Recurring gap.

Directly missed opportunities:
- Telemetry pipelines at Microsoft = distributed counters at scale. Exact same problem.
- Billing at Rippling = idempotency and exact counting under concurrent writes.
- Circuit breaker = standard pattern used in Azure-based distributed services.

This is the biggest gap to close before the Delve interview. Interviewers want to hear "at Rippling we had a similar problem with billing counters, here's what we did and what I'd do differently."

### 8. Communication — 3/5

Generally clear, but required multiple prompts to fully articulate ideas. Trade-off reasoning improved significantly mid-session — started documenting decisions in the diagram, which is good.

### Diagram Quality — 4/5

Strengths:
- All key components present: Client, API Gateway, Load Balancer, Rate Limiter Service, Redis
- Data flow is directional and clear
- Redis key patterns explicitly labeled (config vs counter separation)
- Lua script flow documented inline
- Trade-off note on async replication is excellent — shows architectural thinking

Gaps:
- No circuit breaker shown (introduced by interviewer)
- Only one Rate Limiter Service instance drawn — should show multiple for horizontal scaling
- Token bucket state (tokens + timestamp) not reflected in Redis key schema

### Overall: 3/5 — Conditional Hire at mid-level, needs work for senior

Strong fundamentals, good trade-off reasoning once prompted, clean diagram. The gaps are in proactivity (raising NFRs, resilience patterns, real-world anchoring) rather than knowledge — most answers known once pushed. Close those gaps and this is a strong senior performance.
