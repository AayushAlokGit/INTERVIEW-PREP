# System Design Round Transcript
**Date:** 2026-06-03
**Start Time:** 16:17
**End Time:** 17:02
**Duration:** 45 minutes
**Problem:** Design a distributed in-memory cache (Redis-like)

---

## Conversation Log

**Interviewer:** Design a Redis-like distributed in-memory cache. Start by gathering requirements with numbers.
**Aayush:** Asked what the value can be.
**Interviewer:** Opaque byte blobs, avg ~1KB, cap 256KB; v1 is key→blob only.
**Aayush:** Asked if each SET overwrites (no versioning).
**Interviewer:** Yes — last-write-wins. Now state requirements with numbers.
**Aayush:** FRs: GET, SET, DELETE, TTL/expiry. Out of scope: eviction, rich structures. NFRs: HA 99.99%, GET p99 < 10ms, fault tolerant, 10M daily clients (1 write + 10 reads/day → ~100 writes/s, ~1000 reads/s, peak 10x), 1B keys × (1KB key + 10KB value) = 11TB.
**Interviewer:** Storage math checks out. But 10 reads/client/day is unrealistic — a single backend service hammers its cache constantly. Re-derive read QPS.
**Aayush:** Assume 10 reads/s per client → 100M reads/s avg, 1B/s peak.
**Interviewer:** Big swing — ground it in concurrency (daily ≠ concurrent). If you stand by ~100M reads/s, own the assumption. Now turn it into machines: nodes for throughput (~100-200K ops/s/node) vs nodes for 11TB storage. Which dominates?
**Aayush:** Throughput-bound — ~100 nodes for throughput, fewer for storage.
**Interviewer:** Conclusion right, arithmetic off. 100M ÷ 100K = ~1,000 nodes (not 100); storage ~85 nodes. Throughput dominates ~10x → points at replication, not just sharding.
**Aayush:** Asked to add details to the drawio.
**Interviewer:** Added requirements + capacity notes (no architecture). Now: core entities.
**Aayush:** Key, Value (bytes), Metadata (expiryTime).
**Interviewer:** Good — CacheEntry { key, value, expiresAt }. Now design the API contract — explicit request/response shapes, miss/expiry/error behavior.
**Aayush:** Auth via JWT. GET key → 4xx if absent else value bytes. PUT key {value, ttl?} → add if absent (with ttl), overwrite value if present (keep old ttl), 2xx. DELETE key → success code, no-op if absent.
**Interviewer:** Probed: idempotency (why PUT), TTL-on-overwrite choice, batch reads at 100M/s.
**Aayush:** PUT retry-safe since end result same. Would reset TTL on overwrite. Add a batch query endpoint.
**Interviewer:** Good. Now high-level design — how does a client find the node for key X across ~1,000 nodes?
**Aayush:** Write path: client PUT → load balancer → consistent hashing to node → in-memory KV, periodic flush to disk for crash recovery. Distributes write load. Accepts write loss on node down; could use leader-follower with failover but that adds conflict resolution.
**Interviewer:** Where does consistent hashing actually happen (LB doesn't know keyspace)? And with one leader per key, where does conflict really come from?
**Aayush:** A cluster coordinator holds key→node mapping; client asks it which node to contact. On failover, writes lost for failover duration.
**Interviewer:** Coordinator on every request = 100M lookups/s = recreated bottleneck + SPOF. How to keep it off the hot path, and is mapping per-key or coarser? And: is replication sync or async — can an acked write still be lost?
**Aayush:** Client caches topology and contacts node directly (like Redis). Async replication → eventual consistency, accepted for high availability.
**Interviewer:** Good — that's the Redis Cluster slot model; async = AP. Now the read path: put followers to work for reads? Reconcile 85 shards (storage) vs 1,000 nodes (throughput) — what is N followers per shard?
**Aayush:** (Asked to update drawio.)
**Interviewer:** Drew his described architecture faithfully. Pushed back to the read-path question.
**Aayush:** ~10 followers per shard (90 machines fit data, 1000 needed for throughput). Route reads to leader and/or followers (eventual consistency cost). Helps hot keys — hot-key reads spread across the shard's followers.
**Interviewer:** Quantify the hot-key claim: one key on one shard = ~11 nodes ≈ 1.6M reads/s. What if one key gets 10M reads/s? And the hot WRITE key (all writes to one leader)?
**Aayush:** Replicate hot key across multiple shards — but first need hit-rate counting to detect, then dynamically copy across shards and update client mapping.
**Interviewer:** Hot writes — replicating across shards reintroduces multi-master conflict. Can you scale a hot write key at all? And is there a cheaper hot-read mitigation than resharding?
**Aayush:** Hot write: replicating across shards spreads the meltdown to multiple nodes — better to reduce write count by batching write events. Hot read: clients cache values in local memory.
**Interviewer:** Both correct and honest. Wrapped up.

---

## Design Summary

**Requirements Gathered:**
- FRs: GET, SET (PUT), DELETE, TTL/expiry. Out of scope: eviction, rich data structures.
- NFRs: 99.99% HA, GET p99 < 10ms, fault tolerant, last-write-wins.
- Scale: 1B keys, ~11TB; ~100M reads/s avg / ~1B/s peak (after correction); throughput-bound (~1,000 nodes vs ~85 for storage).

**High-Level Architecture:**
- Client caches topology (slot→node map) from a coordinator; routes GET/PUT directly to the owning shard leader via consistent hashing (Redis Cluster style).
- ~85 shards, each 1 leader + ~10–12 followers (~1,000 nodes total).
- Async replication leader→followers; reads fan out across leader + followers for throughput.
- Periodic disk flush per node for crash recovery.

**Key Design Decisions & Trade-offs:**
- PUT (idempotent) for last-write-wins; retries safe.
- Async replication → AP / eventual consistency, accepted for availability (acked writes can be lost on failover).
- Client-cached topology to keep the coordinator off the hot path.
- Reads from followers trade staleness for throughput.

**Scalability & Fault Tolerance Points:**
- Throughput vs storage node-count reconciliation (N≈10 followers/shard).
- Hot read key: detect via hit-rate counting → replicate across shards, or client-side L1 cache (cheaper).
- Hot write key: hard ceiling; mitigate by batching/coalescing writes (valid under LWW).

**Gaps / Missed Areas:**
- Initial read QPS wildly off (10/day) then over-corrected without grounding in concurrency.
- Arithmetic slip on throughput node count (100 vs 1,000).
- Routing mechanism vague initially ("LB does consistent hashing").
- Coordinator hot-path/SPOF needed prompting.
- Operability under-covered: replication-lag monitoring, stale-read detection, failover-time vs SLA budget, fleet cost.
- Deferred the read-path answer twice via diagram-update requests.

---

## Feedback Given

**Time Taken: 45 minutes**

### Standard Evaluation
- Requirements clarification — Mixed: good clarifying Qs and proactive NFRs, but read-QPS estimate flip-flopped (10/day → 100M/s) without grounding in concurrency.
- Core entities — Strong: clean minimal CacheEntry, no over-modeling.
- API design — Strong: deliberate PUT idempotency with correct retry reasoning, idempotent DELETE, added MGET. Thin GET response shape.
- High-level architecture — Good: real write path, self-raised persistence/replication/failover; routing corrected to coordinator + client-cached slot map.
- Component design & trade-offs — Good: named async vs sync, committed AP; coordinator hot-path needed prompting.
- Scalability & fault tolerance — Strong: reconciled throughput vs storage into ~10 followers/shard, reads across followers.
- Deep dive — Strong: self-raised hot keys, engaged single-shard ceiling, detection + hot-key replication, honest hot-write ceiling, client L1 cache.
- Communication — Mixed: strong reasoning but deferred read-path answer twice via diagram requests.

### Three fixes
1. Ground capacity numbers in concurrency, not vibes; recheck arithmetic (100M/100K = 1,000).
2. Don't defer with the diagram — answer verbally first.
3. Tighten response shapes with explicit fields.

### Senior-signal scorecard
- Own the narrative / self-raise traps — Strong.
- Lead with trade-offs vs named alternatives — Mixed.
- Push scale until it breaks — Strong.
- API as a designed contract — Strong.
- Operability / second-order concerns — Mixed.
- Pace — Strong.

**Level read:** Mid-to-senior, trending up — a hire leaning senior. Self-raising and deep dive were senior; shaky capacity sizing and prompted corrections keep it from a clean strong-hire.

### What a senior strong-hire would have done on THIS problem
- Sized QPS from concurrency first try; computed 1,000 nodes correctly.
- Named client-side slot-map routing immediately and self-raised the coordinator-bottleneck.
- Self-raised async vs sync and acked-write loss unprompted.
- Quantified the hot-key single-shard ceiling themselves; led with client-side L1 cache before resharding.
- Closed on operability: replication-lag metric + stale-read SLO, failover time vs 99.99% budget, fleet cost sanity check.
