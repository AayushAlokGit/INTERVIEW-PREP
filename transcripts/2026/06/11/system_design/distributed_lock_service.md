# System Design Round Transcript
**Date:** 2026-06-11
**Start Time:** ~11:55 (not explicitly captured at round start)
**End Time:** 12:46
**Duration:** ~50 minutes
**Problem:** Design a Distributed Lock Service (Chubby/etcd/ZooKeeper-style)

---

## Conversation Log

**Interviewer:** Design a distributed lock service — many clients coordinate by acquiring/releasing locks on named resources (leader election, single-worker-per-job). Start with requirements.

**Aayush:** Is there a catalog of named resources or do clients supply the name?

**Interviewer:** Clients supply an arbitrary string key; no pre-registered catalog. Lock exists implicitly. Continue requirements.

**Aayush:** Do we need to support TTL for locks?

**Interviewer:** Your call — consider a client that crashes/partitions before releasing. Decide, and anticipate the new problem a TTL introduces.

**Aayush:** Good idea to support TTL so a crashed process doesn't hold a lock forever.

**Interviewer:** You skipped the new problem: client A acquires with 30s TTL, GC-pauses 35s (not dead), TTL expires, lock granted to B, A wakes still thinking it holds it → both write. How to prevent? And should clients renew TTL?

**Aayush:** Use a monotonic fencing token given to the client acquiring the lock first; it's associated with the client; a long-running process refreshes the lock using the token; a new client gets a different token.

**Interviewer:** Good — but the token only protects anything if the downstream protected resource validates it and rejects any write with a lower token than the highest seen. Now consolidate: full FR list, NFRs WITH numbers, and the CAP choice under partition — consistency or availability? Justify.

**Aayush:** FRs: (1) clients acquire locks on named resources; (2) mutual exclusion; (3) TTL; (4) fencing tokens. NFRs: (1) HA for acquire 99.99%; (2) low latency p99 < 100ms; (3) acquire atomic, no 2 concurrent holders; (4) 1M concurrent clients.

**Interviewer:** You listed "highly available" AND "no two holders" — under partition these conflict. If the cluster splits and a client on each side asks for the same lock, available = both granted (broken), consistent = only one side grants, other rejects. Pick one and justify. Also your scale is incomplete — ops/sec? locks held? memory?

**Aayush:** For a lock service consistency is more important to prevent downstream issues — once acquired, all others see it held. Acquire/release ~1M ops/s avg, 10x peak. ~1KB/key → 1GB memory avg, 10GB peak.

**Interviewer:** CP it is — minority partition unavailable by design, correct trade. Numbers noted; 1M writes/s through a consistency protocol will be the tension. Now core entities — key data objects and fields.

**Aayush:** Core entities: (1) Client; (2) Lock {key, fencingToken, clientId, expiry}.

**Interviewer:** Good and minimal. Decision that shapes the rest: when B tries a lock A holds — fail-fast or block/wait? That determines whether you need a wait-queue entity. Then give the API with explicit request/response shapes.

**Aayush:** Identity from auth token. POST /locks Req:{key, expiry} Resp: fencingToken, Idempotency-Key header. PUT /locks/{key} Req:{fencingToken, expiry} Resp: 2xx. DELETE /locks/{key} Resp: status lock deleted.

**Interviewer:** Good — auth-token identity and self-raised idempotency. Three gaps: (1) DELETE has no fencing token → any client can release anyone's lock; (2) POST response doesn't cover "already held" — what status/body? fail-fast vs block still unstated; (3) if blocking, how does the client wait?

**Aayush:** (1) Pass the fencing token on DELETE. (2) POST returns an appropriate status indicating resource already taken. (3) Fail-fast → many retries; block → reduces throughput. For blocking, client has a wait_timeout and a fixed number of retries, implemented client-side; if fail-fast, client throws an exception so the parent process can retry.

**Interviewer:** Reasonable; cleaner blocking = server long-poll with wait_timeout. Now the HLD — components, where lock state lives, how acquire flows. Remember you chose CP — you owe me a mechanism for a single agreed answer under failure.

**Aayush:** Redis cluster — chosen for TTL support and in-memory low latency. Redis gives atomicity via Lua scripts so no two concurrent clients acquire the same lock. For availability, multiple Redis nodes, lock requests distributed via consistent hashing. With multiple nodes, network partitions are inevitable; post-partition the system can be inconsistent and must be brought back to consistency by a coordinating mechanism.

**Interviewer:** You committed to CP, but Redis does NOT give CP. Replication is async: primary acks acquire to A, crashes before replicating, replica promoted (never heard of A's lock), B acquires on new primary → A and B both hold it. Mutual exclusion violated. Does Redis satisfy your CP guarantee? If not, what kind of system do you need — name it and what it does differently.

**Aayush:** Instead of one node we'd have to rely on consensus among the nodes.

**Interviewer:** Make it concrete: (1) name the protocol; (2) name a real system used as a lock/coordination store; (3) what does "commit" mean and why does it make the double-grant impossible — when does the leader ack the client?

**Aayush:** Not very sure about the consensus algorithms.

**Interviewer:** (gives names) Raft/Paxos; etcd (Raft), ZooKeeper (ZAB), Consul (Raft). These run an odd number of nodes; a write is acked only after a majority records it. Reason: (1) in the crash scenario, could A's acquire have been acked at all? (2) on a 3-vs-2 split, which side grants, which blocks, and how does that map to CP?

**Aayush:** (1) Writes complete once majority nodes ack. (2) The 3-node partition can accept writes since it has majority.

**Interviewer:** Correct — A's acquire was never majority-committed so failover doesn't lose it; 3-side grants, 2-side refuses = CP made concrete. Now scale: a Raft group has ONE leader, ~10–30k writes/s. Your NFR is 1M avg / 10M peak, all writes. One Raft group is 50–100× short. Fix it — you used a relevant idea earlier. What new problem does it introduce?

**Aayush:** Consistent hashing of the keyspace and using multiple Raft groups.

**Interviewer:** Right — partition keyspace across many Raft groups. But you skipped the new problem. Single hot lock "leader:cluster-A" — 200k acquire-attempts/s, consistent hashing sends it to ONE group, can't shard a single key. (1) What's it called? (2) What do you actually do about a single hot lock?

**Aayush:** Hot key creating a hot shard. For hot keys, dynamically recognise them and create copies across multiple nodes to distribute read traffic — but this adds complexity keeping copies consistent.

**Interviewer:** Doesn't fit: these are writes (acquire attempts), not reads; and you can't have two writable copies of a lock without breaking mutual exclusion. Reframe — only one client can ever win; once held the rest are waiting. Where's the load coming from and how do you stop 200k pointless writes/s against a held lock? What are losing clients doing, and how does a leader-election client differ from a naive retry loop?

**Aayush:** The losing clients are blocked.

**Interviewer:** Right thread, but "blocked" splits two ways: naive blocking = loop try/sleep/retry (each retry still a write → shard still melts); correct = register a WATCH and go idle (zero writes), service pushes a notification on release/expiry, only then next contender attempts (ZooKeeper ephemeral-sequential + watch). Collapses load to ~one write per ownership change. That's how you tame a hot lock, not replication. Let's stop here.

---

## Design Summary

**Requirements Gathered:**
- FRs: acquire/release/renew locks on client-supplied named keys; mutual exclusion; TTL (crash auto-release); monotonic fencing tokens.
- NFRs: CP under partition (chose consistency, minority unavailable by design); p99 acquire < 100ms; ~1M ops/s avg, 10M peak; ~1M locks held (1GB) up to 10M (10GB) @ 1KB/key.

**High-Level Architecture (as described):**
- Clients consistent-hash onto a Redis cluster; each Redis node uses Lua scripts for atomic SET+TTL.
- (Corrected under questioning toward consensus / Raft, but original choice was Redis.)

**Key Design Decisions & Trade-offs:**
- TTL + fencing tokens to handle crashed/paused holders.
- CP over AP — correct for a lock service.
- Sharding keyspace into multiple consensus groups for throughput (reached under prompting).

**Scalability & Fault Tolerance Points:**
- Recognized partitions are inevitable with multiple nodes.
- Correctly reasoned majority-commit and 3-vs-2 partition behaviour once given consensus vocabulary.
- Applied keyspace sharding / multiple Raft groups to scale writes.
- Identified hot key → hot shard.

**Gaps / Missed Areas:**
- Chose Redis (async replication) despite committing to CP — didn't catch the contradiction himself.
- Blanked on consensus algorithms (Raft/Paxos) — core knowledge gap for this problem.
- Did not self-initiate the scale-break (1M writes vs single Raft leader).
- Wrong hot-key mitigation (replicate for reads) — it's a write-contention problem and locks can't be replicated; needed to be walked to watch/notify.
- No monitoring/alerting, cost-at-scale, or backpressure discussion.
- Did not state a concrete 409 / blocking long-poll cleanly.

---

## Feedback Given

### Standard scorecard
- **Requirements:** Strong-ish — load-bearing clarifying questions (catalog vs client-supplied, TTL), drove TTL → fencing. Missed that "HA + mutual exclusion" conflict under partition until forced; then chose CP correctly.
- **Core entities:** Good — tight Lock model, no waste.
- **API design:** Mixed, trending up — auth-token identity and self-raised idempotency (a recurring weakness, nailed this time). Gaps: DELETE missing fencing token initially, vague "already held" status, fuzzy blocking mechanism.
- **HLD:** Broke here — chose Redis (wrong for CP), didn't catch the contradiction with his own #1 requirement.
- **Deep dive:** Blanked on consensus algorithms (the core gap); recovered well once given names (reasoned quorum + partition correctly unaided). Didn't self-initiate scale-break; applied sharding correctly when prompted. Hot-key mitigation wrong (replication for a write/lock problem); found "block" thread but watch-vs-spin had to be drawn out.
- **Communication:** Solid — honest about not knowing, engaged with every probe.

### Senior-signal scorecard
| Signal | Status | Reason |
|---|---|---|
| Owns the narrative / self-raises traps | Mixed | Self-raised idempotency, fencing, post-partition risk; missed Redis-vs-CP contradiction |
| Leads with trade-offs vs named alternatives | Mixed | Justified Redis & CP; didn't name Redis alternative until pushed; no Raft-vs-Paxos |
| Pushes scale until it breaks | Mixed | Didn't self-initiate; applied sharding correctly once raised |
| API as a designed contract | Mixed | Idempotency + auth-token strong; missing DELETE token, vague 409, fuzzy blocking |
| Operability / second-order concerns | Mixed | Named hot-key→hot-shard but wrong fix; no monitoring/cost/backpressure |
| Pace | Mixed | ~50 min and reached the deep dive — improvement over usual 65–70 overrun |

**Overall: mid-level, "hire" — not yet senior strong-hire.** Gating issue = consensus knowledge gap; a senior opens with "etcd/Raft because writes commit on majority quorum." Reasoning once equipped was clean; pace and idempotency instincts the best seen so far.

**What a senior strong-hire would have done on THIS problem:**
1. Chosen the CP store first and named it (etcd/Raft / ZooKeeper), explicitly rejecting Redis for async-replication lock loss.
2. Self-raised the full TTL→fencing→resource-enforcement chain (resource must reject stale tokens).
3. Pushed own scale: one Raft group ~10–30k writes/s → 50–100 range-partitioned Raft groups (etcd/TiKV model) + rebalancing + hot-shard handling.
4. Tamed the hot lock with watches/notifications (ZooKeeper ephemeral-sequential + watch) + backoff + lease-based leader election — not replication.
5. Owned operability: per-key write-rate metrics + alerts to detect hot shards, fleet cost of 100 five-node groups, blast radius of a leader-down range (unavailable under CP, acceptable).

**Drill:** guidance signals #1 and #3 (own the narrative, push scale); close the Raft basics gap (leader, log replication, majority commit, election; etcd/ZooKeeper/Consul).
