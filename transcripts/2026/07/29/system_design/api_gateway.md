# System Design Round Transcript
**Date:** 2026-07-29
**Start Time:** 15:11 · **End Time:** 15:59 · **Duration:** 48 min
**Problem:** API Gateway
**Performance Rating:** 2/5  <!-- machine-read on future rounds; ≤3 = eligible for re-ask, ≥4 retired -->

## Phase Timings
| Phase | Budget | Actual | Hit? |
|---|---|---|---|
| Requirements | 8 min | 12 min | ✗ |
| Core entities | 12 min | 16 min (names only, no fields) | ✗ |
| API design | 17 min | never completed | ✗ |
| High-level design | 27 min | 31 min | ✗ |
| Deep dive | 40 min | 31→42 min (11 min) | ✗ |

---

## Conversation Log

**Interviewer:** Design an API Gateway. A company runs a few hundred backend microservices; every external client — mobile, web, third-party partners — enters through a single front door. It sits between the public internet and the internal service mesh and is the only thing those clients can reach. Start by gathering requirements — what it needs to do, what you're explicitly leaving out, and numbers on the non-functional side.

**Aayush:** what all functionalities does the API gateway serve is it just a proxy for the internal service mesh?

**Interviewer:** Largely your call — that's the design decision. Context: it's not a dumb reverse proxy (else we'd use nginx); clients are heterogeneous including third-party partners who are metered and contractually rate-limited; backend services are owned by many teams deploying independently several times a day. What's in scope, and what would you explicitly keep out?

**Aayush:** can i assume 100M DAU for the system and assume each user first 5 requests per day?

**Interviewer:** Assume what you can defend. 100M DAU fine — but 5 requests/user/day, think about what a mobile app does in one session. Give me average RPS with the arithmetic, your peak and the multiplier, what the gateway adds to every request and your budget for that, your availability target, and the in-scope/out-of-scope list.

**Aayush:** FRs: 1. Sits between public internet and internal service mesh 2. Redirects requests to appropriate service 3. Authentication of the request 4. Rate limiting of incoming requests. NFRs: 1. Low latency rate limiting (p99 < 10ms) 2. Low latency authentication (p99 < 10ms) 3. Highly available (99.99 ~ 52mins downtime/year) — when API gateway down reject requests 4. Low latency forwarding of requests to appropriate service 5. Assuming 100M DAU each making 50 requests/day → 50k requests/s avg peak 10x.

**Interviewer:** Three things. (1) 100M × 50 = 5B/day ÷ 86,400 ≈ 58K RPS, not 50K — and 10× peak is aggressive; real diurnal is 2–4×. Defend or revise. (2) "99.99%, when down we reject" describes being down, not a target. Every request goes through this box, so its availability is a ceiling on every service behind it — what does that imply about the availability of the components the gateway itself depends on for auth and rate limiting? (3) You have routing, authn, rate limiting; you haven't used two things I told you — partners are metered, and teams deploy several times a day. And I've asked twice for the explicit out-of-scope list. Also: how much latency is the gateway allowed to add end-to-end?

**Aayush:** 1. peak is 5x and assuming 10^5 seconds in a day 2. The availability demand for the API gateway needs to be higher 99.999 since this is the core dependency and will set the upper bound for the entire system availability. 3. The api gateway should support versioning of API endpoints

**Interviewer:** 10^5 s/day is a fine approximation and 5× is defensible, so 50K avg / 250K peak stands. Your availability reasoning is right and is the strongest thing you've said. But note: I asked what 99.999% implies about the gateway's *dependencies*, and you answered why the gateway needs it. If auth requires an identity-store call and rate limiting requires a counter store, those are now in the request path and a 99.99% dependency caps you at 99.99%. Still unanswered: out-of-scope list, partner metering, total latency budget. Locking requirements. Core entities — name them with the fields that matter.

**Aayush:** Core Entities → 1. RateLimitingPolicy 2. Request URL to service mapping

**Interviewer:** Names, not entities — no fields on either, and fields are the point because they determine lookup cost. RateLimitingPolicy keyed by what — user, API key, partner org, endpoint, a pair? That changes cardinality by orders of magnitude and decides whether it fits in node memory. URL→service: keys on path only, or path + method + version + host? Exact match or prefix/pattern? At 250K RPS that's a hash lookup vs a trie walk. And who is making the request? Rate limits and metering are per-caller and you have no caller entity.

**Aayush:** help me out with the core entities and API endpoints

**Interviewer:** Can't — the API contract is what I'm assessing most directly. A wrong, reasoned attempt scores; asking me to fill it in doesn't. Narrowing it: (1) `POST /v2/payments/charge` with an Authorization header arrives — what does the gateway extract first? Those are your fields. (2) To decide where it goes, what's the lookup key and what object comes back, with its fields? (3) To decide if the caller is allowed, same question. (4) The gateway has its own control-plane API — a team deploys at 3pm and needs the gateway to route to it. Verb, path, body. Who's allowed to call it?

**Aayush:** 1. client from auth headers 2. Key of lookup → version of API(v2) + API endpoint (payments/charge). Object returned → Service details of the v2 endpoint including the ip address of service 3. API gateway makes request to auth service, with endpoint and clientId and gets whether client allowed or not 4. Gateway needs to maintain a mapping of endpoint route to service details, updated every time the BE team makes a new deployment. This would be a POST endpoint and the body would be the new API endpoints added to the service manifest.

**Interviewer:** (1) Teams deploy several times a day and instances autoscale — if the route table stores an IP, what happens at 3:05pm when that pod is gone? What should the value hold, and who resolves it? (2) Your key is version + path; `GET` and `POST /v2/payments/charge` collide. Intentional? (3) The big one: you're making a network call to an auth service on every request — a round-trip inside p99 < 10ms at 250K peak RPS, and that service is now a ceiling on your 99.999%. Does authentication actually require a network call? What property would a credential need for local validation, and what do you give up? (4) Control-plane endpoint still vague: verb, path, body fields, response, **who is authorised** (this endpoint can redirect all payment traffic to an attacker), and how a change reaches hundreds of nodes.

**Aayush:** 3. JWT in auth header will contain information about the user and the token audience. The JWT token will be generated in the client side by making a call to the authentication service, so the token would be generated only if the client would have permissions for the requested resource. So the API gateway just needs to verify if the JWT token is correct and valid which will be in memory verification. 4. *(blank)*

**Interviewer:** Right move, and reasoned rather than guessed. Now the senior part — what did you give up? An admin revokes a partner at 14:00 and the token is valid until 14:30; acceptable? And verification needs the issuer's public key — where from, what on rotation, is that a network call you just reintroduced? One line each, then go straight to the HLD and fold in the control-plane path: two flows on one diagram.

**Aayush:** the client makes the request which arrives to the API gateway. Now the API gateway will perform the local verification of JWT key to extract the client details. The client details will be used to check whether the current request is allowed by the rate limit or not. For this contact with an external rate limiting system which is out of scope for this design. Once decided that it is okay to allow a request then the API gateway will check an external service explorer to get the ip address of the service which needs to be targeted. When the devs are deploying a new version of the service they will update the service explorer with the new details so that the API gateway can learn of the new deployment.

**Interviewer:** Look at what you drew. You took the auth service out of the hot path — no round-trip, no ceiling. Then in the same design you put two per-request network calls back in: the rate limiter and the Service Explorer. Both hard dependencies, both cap 99.999%. The argument you made applies to them unchanged and you didn't apply it. And rate limiting is your own FR #4 with your own p99 < 10ms NFR — declaring it out of scope removes it from the part you're willing to defend, not from the design. Deep dive: partners limited to 10,000 req/min per API key, hundreds of nodes behind an LB. (1) A partner's requests land on 50 nodes; how does any node know the global count? (2) If the answer is Redis: 250K increments/sec in a 10ms budget, and it can take the gateway down — size it, then fail open or fail closed? (3) Alternative: each node enforces 10,000 ÷ N with no coordination — what does that buy and what does it get wrong, with the input where it behaves badly? (4) Is there something in between?

**Aayush:** 1. have a centralised counter store so that multiple nodes see the same thing use redis. 2. When redis fails we fail open, let traffic through to allow for service to still be provided. We don't deny service just because a rate limiting dependency failed. We could also apply this rule depending on the actual endpoint and have fail open or closed depending on the actual endpoint. 3. With each node maintaining its own quota of the overall quota limit there is no need for coordination between nodes, this buys latency which would have come in if redis was present. But this also causes no coordination between the nodes, requests could be sent to a node which has already exhausted its quota while other nodes with active quota still sit. Requests would have to be routed based on the quota present for each node which would again bring in extra network calls. 4. *(blank)*

**Interviewer:** Fail-open with the policy varying per endpoint is a genuinely senior distinction. And you named the real defect in local quotas — fragmentation, a partner rejected at 6,000 global requests because traffic landed unevenly. But you didn't size Redis, and question 4 is blank — third time the last question in a list went unanswered. Three minutes: you don't have to choose between coordinating per request and never coordinating. What if nodes coordinate on a schedule — what do they exchange, what does each do between exchanges, and what does the guarantee become?

**Aayush:** nodes can sync their local counts to redis periodically

**Interviewer:** Right shape, no mechanism and no error bound. Time — feedback.

---

## Design Summary

**Requirements Gathered:**
- FRs: sits between internet and service mesh; routes to the appropriate service; authenticates the request; rate limits; versioning of API endpoints.
- NFRs: rate limiting p99 < 10ms; authentication p99 < 10ms; availability 99.999% (correctly argued as a ceiling on all downstream services); low-latency forwarding.
- Scale: 100M DAU × 50 req/day = 5B/day ÷ 10^5 s ≈ **50K RPS avg**, 5× peak = **250K RPS**.
- Never produced: out-of-scope list (asked 3×), partner metering, total gateway latency budget.

**High-Level Architecture:**
Client → API Gateway (local JWT verification → extract client details → rate-limit check → route lookup) → Backend Service. Gateway calls an external rate-limiting system (declared out of scope) and an external Service Explorer to obtain the target service IP. Devs update the Service Explorer on deployment.

**Key Design Decisions & Trade-offs:**
- **Local JWT verification instead of a per-request auth-service call** — self-derived from the availability-ceiling argument. The strongest moment of the round. Costs (revocation window, JWKS key rotation) never named despite a direct ask.
- **Fail open on rate-limiter failure, with the policy varying per endpoint** — genuinely senior nuance.
- **Central Redis counter vs per-node local quota** — trade-off articulated correctly; fragmentation named as the cost of local. No concrete input given, no sizing, no middle-ground design.

**Scalability & Fault Tolerance Points:**
- 250K peak RPS established.
- Redis as a hot-path dependency identified by the interviewer, not self-raised; never sized.
- Hot-key concentration on a single heavy partner never surfaced.
- Config propagation to hundreds of nodes never designed.

**Gaps / Missed Areas:**
- **API contract absent.** No paths, request bodies, response shapes, status codes, pagination, or idempotency anywhere in the round. Control-plane endpoint named as "a POST" with an unspecified body; authorisation on it never raised even when bolded in the question.
- **Entities had no fields**, asked twice — which is why Redis sizing was not computable later.
- **IP address stored in the route table** despite services deploying several times daily; flagged and moved past.
- **Own principle not applied consistently** — removed one hot-path dependency, added two.
- **Partner metering** present in the prompt, never picked up.
- **Three trailing questions unanswered** (JWT costs, Redis sizing, Q4) plus the out-of-scope list.
- **One explicit hint request.**
- Monitoring, hot partitions, cost at scale: untouched.

---

## Feedback Given

**Time Taken: 48 minutes.**

The deep dive was the best part of the round, and it was reached with nine minutes left. Three genuinely senior moments — deriving local JWT verification from the availability argument, fail-open with per-endpoint policy, and naming quota fragmentation as the cost of local counters — all occurred after minute 30 in a phase that got half its budget.

The API contract never happened. Fourth round running, and now the single clearest thing between him and a senior signal. After 48 minutes there is still no path, request body, response shape, status code, or authorisation rule for the one endpoint he named. The control plane of a gateway is not a side detail — an unauthenticated route-update endpoint lets anyone redirect all payment traffic to a host they own; he never raised authorisation even when it was bolded.

Entities got names, never fields, asked twice, because cardinality decides whether the policy set fits in node memory and whether route lookup is a hash or a trie. The Redis sizing failed later precisely because that cardinality was never established.

The costliest pattern: he argued correctly that the auth service must not sit in the hot path, then drew a design with two per-request network calls in the hot path without noticing he'd violated his own principle. That is the difference between having an insight and holding a principle.

Storing an IP in the route table was flagged and passed over; the value needs to be a service identity resolved at call time. Three trailing questions went unanswered plus the out-of-scope list asked three times, and there was one explicit hint request.

### Senior-signal scorecard
| Signal | Status | Reason |
|---|---|---|
| Owns the narrative | Weak | Three trailing questions unanswered, out-of-scope asked 3× and never given, one direct hint request. |
| Leads with trade-offs vs alternatives | Mixed | Fail-open vs closed and local vs global both defended well; Redis, Service Explorer and JWT chosen with no named alternative. |
| Pushes scale until it breaks | Weak | Never sized Redis despite a direct ask; never volunteered a break — it was handed to him. |
| API as a designed contract | Weak | No paths, bodies, shapes, status codes, pagination or idempotency. Control-plane authorisation never raised. |
| Operability / second-order | Weak | Fail-open was real; nothing on monitoring, hot partners, config propagation, or cost. |
| Pace | Weak | Deep dive began at minute 31 with 9 minutes left; requirements + entities took 16 minutes and produced no fields. |

**Overall read: mid-level. No-hire at a senior bar for this round** — not because the thinking is weak (the deep-dive reasoning was the opposite) but because the contract phase produced nothing assessable and the design had to be extracted rather than presented.

### Performance Rating: 2/5 — Weak. Eligible for re-ask.

### What a senior strong-hire would have done
- Self-raised the hot-path principle as a **rule** and audited every dependency against it: auth local via JWT; rate limiting therefore cannot be a per-request round-trip either; routing config **pushed** to nodes and held in memory so the request path touches zero external services.
- Given the control plane a real contract: `PUT /admin/v1/routes/{routeId}` with `{serviceName, pathPattern, methods, version, timeoutMs, retryPolicy}`, an explicit response, mTLS/service-identity authorisation, versioned and rollback-able changes; nodes watch the config store and cache in memory.
- Pushed the scale break themselves: at 250K RPS the interesting break is not Redis throughput but a **single hot partner concentrating on one counter key**. Fix: key-splitting (`partner:{id}:{shard}` with quota divided across shards), or local token buckets leasing a slice of quota refreshed every ~100ms, with bounded overshoot stated as a number (`N_nodes × leaseSize`).
- Named the JWT costs unprompted: no revocation before expiry → short-lived tokens (~5 min) or a pushed revocation bloom filter; JWKS cached with long TTL and refreshed in the background, never in the request path.
- Covered **metering** — the prompt said partners are metered; every request emits an async usage record to a billing pipeline, fire-and-forget to a local buffer, never blocking the response.
- Reached the deep dive by minute 25 with a real API contract behind them.
