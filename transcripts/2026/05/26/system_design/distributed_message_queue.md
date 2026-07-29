# System Design Round Transcript
**Date:** 2026-05-26
**Start Time:** 12:45
**End Time:** 14:03
**Duration:** 78 minutes
**Problem:** Design a Distributed Message Queue (Kafka-like)

---

## Conversation Log

**Interviewer:** Presented problem, set up draw.io file, asked for requirements (FRs + NFRs with numbers).

**Aayush:** "kafka is like an event store right so producers put events into partitions and then consumers consume from the partition"

**Interviewer:** Redirected — gather requirements, don't mimic existing system. Asked for functional, non-functional, numbers.

**Aayush:** Asked what "topic management" meant.

**Interviewer:** Explained control-plane ops (create/delete/configure topic).

**Aayush:** "what should be the scale for the system?"

**Interviewer:** Pushed back — propose your own scale based on a target use case.

**Aayush:** [updated drawio with FRs (push/pull/topic management/at-least-once) and NFRs (99.99% HA, 30-day durability, p99<100ms, 1M msgs/s avg, 10M peak, 1MB avg msg size)]

**Interviewer:** Bandwidth = 1M × 1MB = 1 TB/s — sanity check. Suggested realistic msg sizes are KB-range. Added ordering, replay, consumer-group FRs to revisit.

**Aayush:** Updated to 500KB.

**Interviewer:** Still 500 GB/s. Pushed for realistic 1-10KB.

**Aayush:** "5KB, 5GB/s peak peak 50GB/s, storage 50GB×86400×3 = 15 PB/s for 3 days"

**Interviewer:** Corrected — storage uses avg not peak, units are PB not PB/s. ~1.3 PB raw × RF 3 = ~4 PB.

**Interviewer:** Moved to HLD — asked for components.

**Aayush:** Updated APIs with messageId/partition/offset response, idempotency-key header, offset/limit consume, partition+replication factor on topic create. Then described: LB routes to broker via consistent hashing, ZooKeeper coordinates, WAL on disk + in-memory queue, broker cluster = topic, consumers via LB.

**Interviewer:** Probed 3 mental-model issues:
1. "Broker cluster = topic" is wrong — cluster hosts many topics, broker hosts many partitions
2. "In-memory queue alongside WAL" — log IS the queue, use OS page cache
3. Consumers don't go through LB — they go direct to leader after metadata lookup

**Aayush:** Corrected #1. Pushed back on #2: "why would in-memory queue break replay if we have offset counter?"

**Interviewer:** Walked through the math — 1.3 PB doesn't fit in RAM, dual storage = 2× write amp, log-as-queue unifies storage + replay + offsets.

**Aayush:** Accepted. For #3: "Consumers can hit control plane with topic details and the control plane can provide the broker address for the partition, consumer connects directly".

**Interviewer:** Moved to durability/replication deep dive.

**Aayush:**
1. Write path: producer → leader WAL → ack returned → async replication. "Replication is for durability not read load so eventual consistency is fine."
2. Async replication, push from leader.
3. Failure: ZK promotes follower with least lag, accept loss.

**Interviewer:** Pointed out contradiction — ack=1 + accept loss directly violates NFR2 ("messages should not be lost") and FR4 (at-least-once). Listed ack=0/1/all options. Noted Kafka uses follower-pull not push. Noted ISR-based election prevents loss.

**Aayush:** "go with acks=all"

**Interviewer:** Confirmed within p99 budget. Moved to consumer-group + sizing deep dive.

**Aayush:**
- A1: Partition assignment via consistent hashing
- A2: "messages in the partition being processed by the consumer will now be processed by some other consumer"
- A3: Offsets stored in "WAL of leader"
- B1: 500 partitions for peak
- B2: 200 brokers

**Interviewer:** Confirmed B1, B2. Glossed-over rebalance mechanics (heartbeat, group coordinator, generation ID, pause-the-world). A3 wrong — pushed for "where could you reuse existing durable infra?"

**Aayush:** "another topic for offsets?"

**Interviewer:** Correct — `__consumer_offsets`, log-compacted, keyed by (group, topic, partition). Reuses replication machinery.

**Aayush:** Asked interviewer to draw the diagram and fix the APIs section.

**Interviewer:** Drew system (producer, LB control-plane-only, ZK control plane, 3 brokers each with P0/P1/P2 leader/follower split, OS page cache layer, consumer group, __consumer_offsets) and fixed APIs (added consumer-group context to offset commit, response shapes on every endpoint, consumer-group join + heartbeat endpoints).

---

## Design Summary

**Requirements Gathered:**
- FRs: produce, consume, topic management, at-least-once, per-partition ordering, replay, one-consumer-per-partition-per-group
- NFRs: 99.99% HA, 3-day durability, configurable RF, p99 < 100ms, 1M avg / 10M peak msgs/s, 5KB avg msg

**High-Level Architecture:**
- Producer → LB (control plane only) → ZooKeeper for metadata bootstrap
- Producer → broker leader directly for publish (acks=all)
- Broker cluster: many brokers hosting many partitions from many topics; each partition replicated RF=3 with leader + 2 followers
- Storage: WAL on disk per partition; OS page cache for hot reads
- Consumer → CP for metadata + group coordination → broker leader directly for reads
- Consumer offsets stored in internal `__consumer_offsets` (compacted) topic

**Key Design Decisions & Trade-offs:**
- acks=all for durability (chose only after contradiction pointed out)
- Log as queue (no separate in-memory buffer) — chose after objection answered
- ISR-based leader election to prevent committed-message loss
- 500 partitions, 200 brokers (sizing correct)

**Scalability & Fault Tolerance Points:**
- Partition fanout for throughput
- 3× replication via ISR
- ZooKeeper for coordination + failover

**Gaps / Missed Areas:**
- Rebalance protocol mechanics (heartbeat, generation ID, group coordinator, partition assignment strategies)
- Unclean leader election trade-off
- Rack/AZ-aware replica placement
- Log segments + retention + cleanup (compaction vs deletion)
- Producer batching, compression
- Schema registry / message format
- Multi-tenant isolation, quotas, ACLs
- Cross-DC replication (mirroring)
- Did not author the architecture diagram independently

---

## Feedback Given

### Scoring (out of 5)

| Category | Score | Notes |
|---|---|---|
| Requirements clarification | 3 / 5 | Mimicked Kafka first; couldn't propose scale independently; multiple BoE corrections. |
| High-level architecture clarity | 3 / 5 | Right components, wrong relationships (broker=topic, in-memory queue, consumer-via-LB). |
| API design clarity | 3.5 / 5 | Improved during round; offset commit lacked group context. |
| Component design & trade-offs | 3 / 5 | ack=1 + "accept loss" contradicting own NFR. |
| Scalability & fault tolerance | 3.5 / 5 | Sizing correct; replication mechanics push vs pull confused; missed unclean election discussion. |
| Deep dive quality | 3 / 5 | Heavy prompting throughout; consumer offsets wrong until hint; rebalance glossed over. |
| Communication | 3 / 5 | Single-line answers needed expansion; diagram lagged verbal. |

### Highlights
- Strong instinct to push back on in-memory-queue assertion
- Sizing math correct once computed
- All key FRs/NFRs landed eventually

### Areas to work on
- Requirements-first not pattern-first
- Reconcile choices against own NFRs (ack=1 vs no-loss contradiction)
- BoE math discipline + units (PB vs PB/s, peak vs avg)
- Diagram-as-you-go
- Time management — 78 min vs 45-60 target

### Diagram Quality (final state)
- All major components present, leader/follower marked
- Data flow directional and labeled
- BUT: diagram authored by interviewer at Aayush's request, not built incrementally by him — recurring weakness

**Time Taken: 78 minutes**
