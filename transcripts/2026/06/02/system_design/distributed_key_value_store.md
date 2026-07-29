# System Design Round Transcript
**Date:** 2026-06-02
**Start Time:** 16:13
**End Time:** 17:33
**Duration:** 80 minutes
**Problem:** Design a Distributed Key-Value Store (Dynamo/Cassandra-style)

---

## Conversation Log

**Interviewer:** Design a distributed key-value store (DynamoDB/Cassandra/Redis-as-a-store). Clients can put(key, value) and get(key). Runs across many machines, survives node failures, scales horizontally. Start by driving requirements with numbers.

**Aayush:** FRs — (1) Put a key,value pair. (2) Get the value for a key. Out of scope — (1) TTL for keys, (2) cache eviction.

**Interviewer:** Is delete in scope? Does put overwrite? What's the value shape/size? Now drive NFRs with numbers — scale, QPS, latency, availability, and especially the consistency model.

**Aayush:** FRs updated — put overwrites if key exists (value can be JSON string); added delete. NFRs: (1) Highly available 99.9% (~8.6 hrs/yr). (2) Eventual consistency acceptable, option for read-your-own-writes. (3) Durable, survive crashes. (4) Low latency p99 < 10ms. (5) 100M services, 1 write/day + 10 reads/day each ≈ 1k w/s, 10k r/s avg, peak 10x → 100k r/s. (6) 1B keys × 1KB avg = 1TB total.

**Interviewer:** Math checks out. Two sharpeners: (a) is data in RAM, disk, or both given the durability NFR? (b) Are you explicitly choosing AP over CP, and why?

**Aayush:** Data lives on disk; also keep in memory to reduce query latency and regularly flush to disk.

**Interviewer:** Good (memtable + on-disk pattern). Now answer the AP-vs-CP question explicitly with the trade.

**Aayush:** Choosing availability over consistency to retain service during network partitions. This introduces complexity of conflict resolution — could use vector clocks (Dynamo-style) to resolve conflicts.

**Interviewer:** Correct — and note consensus (Paxos/Raft) is for CP; in AP you avoid consensus on the write path and reconcile after with conflict resolution. Now core entities.

**Aayush:** Core entities — Key, Value, Metadata (vector clocks).

**Interviewer:** Now the API contract for put/get/delete — verbs, request shapes, response shapes with explicit fields, and how the vector clock flows.

**Aayush:** (1) PUT /key, body {value:string}, returns 2xx, add or overwrite. (2) GET /key returns "value" string, 4xx if missing. (3) DELETE /key, returns appropriate HTTP status.

**Interviewer:** Underspecified. GET returns only a string — where does the metadata/vector clock go? Walk through: client reads X, two clients update X — how does the server know which version a PUT is modifying?

**Aayush:** GET returns {value, vectorClock}; client uses vectorClock for conflict resolution under concurrent updates, or ignores it if none. PUT request carries the vectorClock obtained from the prior GET.

**Interviewer:** Correct (Dynamo context pattern). Tuck away: concurrent writes can produce sibling versions that a later GET returns together for client reconciliation. Now high-level architecture — how does a request find the node owning a key across 1TB?

**Aayush:** Clients hit KV nodes via a load balancer using consistent hashing. Nodes mapped to a hash ring; key mapped to ring, move clockwise to find owning node. Replicate with factor N by continuing clockwise to pick N-1 more nodes. Track the preference list of N nodes per key, first is coordinator. (GET description tangled with write/replication.)

**Interviewer:** GET description got tangled with the write path — separate read and write paths precisely. Also: how many replicas must ack a write (W) and a read (R)? Tie to the consistency promise.

**Aayush:** Read path — if key present on node return value+vectorClock; else check the preference list (populated via gossip) and route to coordinator. Quorum — more than half of N must acknowledge.

**Interviewer:** That majority quorum contradicts your AP choice. N=3, partition leaves coordinator reaching only 1 replica, client PUTs — majority insistence = reject = CP. Pick concrete R, W; does R+W>N hold; resolve the partition while staying AP.

**Aayush:** R = nodes for a successful read, W = nodes for a successful write, N = replication factor. For our AP store, R=1 and W=1 — reads/writes need only one node to ack, but this needs background conflict resolution for eventual consistency.

**Interviewer:** Legitimate AP corner. Now Hole 1 — durability/propagation: (a) the one acked node crashes before propagating — write lost; consistent with the durability NFR? (b) what mechanism propagates to the other N-1 and heals down replicas?

**Aayush:** Anti-entropy with Merkle trees to resolve conflicting keys; hinted handoff where a node stores data for nodes it needs to forward to.

**Interviewer:** Right tools, but you dodged 1(a): with W=1 a crash before propagation loses the committed write, contradicting durability. Confront it — does W=1 satisfy durability, and what do you change?

**Aayush:** Immediately flush the write to disk.

**Interviewer:** WAL-before-ack handles process crash/restart. Residual: W=1 = one durable copy until replication catches up; disk loss in that window still loses it. Now push scale: (1) a node holding part of 1TB dies — who absorbs its range? (2) you add a node — how much moves and is it even?

**Aayush:** (1) The next neighbour absorbs the load and its load doubles; use virtual nodes (one physical node → many ring points) for better distribution on failures. (2) ~1/N data movement, spread evenly across nodes.

**Interviewer:** Correct. But vnodes fix keyspace skew, not a single hot key. One key getting 50k reads/s maps to one set of N replicas that melt. How do you serve a read-hot key? A write-hot key at 50k w/s? And how do you detect it?

**Aayush:** (1) Read-hot — add a random number [1:N] to the key to spread it across nodes and even out load. (2) Write-hot — fan out the write to multiple nodes and aggregate them.

**Interviewer:** Both hand-wavy. Salting forces write-fanout to all variants and needs a read rule — isn't a cache simpler for read-hot? Write-hot "aggregate" only works for mergeable/CRDT values, not an opaque JSON blob — what really happens? And you didn't answer detection.

**Aayush:** We can track the hit count for each key, but this adds complexity of tracking the hit count.

**Interviewer:** Reasonable start; usually done with sampling/heavy-hitter sketches at the coordinator rather than exact counts. Time — let's wrap and debrief.

---

## Design Summary

**Requirements Gathered:**
- FRs: put (overwrite, JSON value), get, delete; out of scope: TTL, eviction.
- NFRs: 99.9% availability; AP over CP; eventual consistency + optional read-your-writes; durable across crashes; p99 < 10ms reads; ~1k w/s, ~10k r/s avg, peak ~100k r/s; 1B keys × 1KB = 1TB.

**High-Level Architecture (verbal):**
- LB → KV nodes; consistent-hashing ring; replication factor N via clockwise preference list; first node = coordinator; gossip for membership/preference list; in-memory layer + WAL/flush to disk.

**Key Design Decisions & Trade-offs:**
- AP over CP; conflict resolution via vector clocks (context echoed through GET→PUT).
- R=1/W=1 for max availability, accepting eventual consistency + background reconciliation.
- WAL-before-ack for durability against process crash.
- Virtual nodes for balanced rebalancing on node add/remove/failure.
- Hinted handoff for transiently-down replicas; Merkle-tree anti-entropy for replica healing.

**Scalability & Fault Tolerance Points:**
- Node death → neighbor doubling, fixed by virtual nodes.
- Node add → ~1/N movement, evenly spread via vnodes.
- Hot-key mitigation attempts: key-salting (read-hot), fan-out/aggregate (write-hot).

**Gaps / Missed Areas:**
- API first draft fieldless; no versioning/error-body/path-vs-body specifics.
- Did not self-raise the AP/quorum contradiction or the W=1 durability hole.
- Write-hot opaque-blob case left hand-wavy ("aggregate" only valid for CRDTs).
- Operability weak: hot-key detection an afterthought; no monitoring/alerting, replica-lag/freshness, or cost story.
- No actual architecture diagram drawn (only text notes for FRs/NFRs/entities/API).
- Pace: 80 min vs 45–50 target.

---

## Feedback Given

### Standard Feedback
- **Requirements clarification — Strong.** Clean scope; self-derived, correct NFR math; explicit AP-over-CP choice with justification.
- **Core entities — Strong.** Key/Value/Metadata identified before API; understood why metadata exists.
- **API design — Mixed.** Vague first cut (no fields, no vector clock); corrected to {value, vectorClock} + PUT context echo under prompting. Missing versioning/error semantics.
- **High-level architecture — Strong verbally, Weak diagram.** Correct Dynamo skeleton produced largely unprompted; never diagrammed; read/write paths initially tangled.
- **Component design & trade-offs — Strong.** R/W/N, R=1/W=1, WAL-before-ack, vnodes, hinted handoff, Merkle anti-entropy — right mechanism each time.
- **Scalability & fault tolerance — Mixed→Strong.** Node-death/rebalancing clean and self-driven; hot keys thinned out.
- **Deep dive quality — Mixed.** Good mechanism recall; needed prompting for every break→fix step.
- **Communication — Mixed.** Correct when prompted; thin first drafts; over time budget.

### Senior Readiness Debrief

**Senior-Signal Scorecard**
| Signal | Rating | Reason |
|---|---|---|
| Owns the narrative / self-raises traps | Mixed | Self-raised vector clocks; did not self-raise W=1 durability, AP/quorum contradiction, or hot keys. |
| Leads with trade-offs vs named alternatives | Mixed | Strong on AP-vs-CP and W=1 cost once pushed; rarely volunteered up front. |
| Pushes scale until it breaks | Mixed | Handled node death/rebalancing well; stopped at comfortable case, waited for escalation to hot keys. |
| API as a designed contract | Mixed | Fieldless first draft; corrected under prompting. |
| Operability / second-order concerns | Weak | Hot-key detection an afterthought; no monitoring/alerting/cost story. |
| Pace | Weak | 80 min vs 45–50; thin first answers forced extra round-trips. |

**Overall level read:** Solid mid-level knocking on senior — **hire, not strong-hire**. Senior-level content knowledge (full Dynamo toolkit), but the design has to be extracted rather than self-driven.

**What a senior strong-hire would have done on THIS problem:**
- Self-raised the W=1 durability contradiction and stated WAL-before-ack + residual disk-loss exposure + "W=2 if bar is higher" unprompted.
- Stated R+W relationship while choosing the numbers ("AP → R+W ≤ N, R=1/W=1, paid for with read-repair + anti-entropy") instead of getting caught in a majority-quorum contradiction.
- Hit hot keys unprompted; cache-first for read-hot; honesty that a write-hot opaque blob has no free fix (coalesce/batch or push back on data model / expose CRDT).
- Owned detection/monitoring: count-min sketch / heavy-hitter sampling at coordinator, per-key QPS metrics + alerts; replica-lag/hinted-handoff-backlog as the staleness signal.
- Drawn a clean ring diagram with preference list, coordinator, labeled read/write paths.
- Finished in ~50 min by pushing first answers to precision before sending.

**Self-drill:** Review system_design_senior_guidance.md, focus on #1 (self-raise traps) and #5 (operability). Drill: after every component, state the trade AND how you'd monitor/detect its failure before moving on.

### Diagram Quality
- The diagram captured FRs, NFRs, core entities, and the API contract as text notes (maintained on Aayush's request).
- **No architecture diagram** was produced — no client/LB/coordinator/replica-ring boxes, no read/write path arrows. All architecture stayed verbal.
- Consequence: the tangled read-vs-write path would likely have been caught immediately had it been drawn; drawing also communicates the preference-list/coordinator structure far faster than prose.
- Recommendation: draw the ring + preference list + labeled read and write paths next time; use the act of drawing to surface gaps.
