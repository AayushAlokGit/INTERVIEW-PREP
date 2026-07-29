# System Design Round Transcript
**Date:** 2026-05-30
**Start Time:** 14:59
**End Time:** 16:10
**Duration:** 71 minutes
**Problem:** Design a Chat / Messaging System (WhatsApp-like)

---

## Conversation Log

**Interviewer:** Please note the time. Design a chat/messaging system supporting 1:1 and group messages, real-time delivery when online, reliable delivery + storage when offline, read receipts, presence. Draw.io path provided.

**Aayush:** 14:59

**Interviewer:** Start with requirements gathering. Lead it.

**Aayush:** Is there any limit on the number of participants in a group?

**Interviewer:** Max 500 participants per group.

**Aayush:** Does the system need to store users' messages or only until delivered?

**Interviewer:** Store full message history (server-side, durable, scrollable across devices).

**Aayush:** What could the scale be — 10M DAU safe?

**Interviewer:** Higher — 500M DAU. You derive the rest: writes/sec avg+peak, storage/year, concurrent online users, read traffic.

**Aayush:** FRs:
1. Users can send 1:1 and group messages (max 500 users) to other users
2. Reliable message delivery irrespective of recipient online status
3. Support read receipts
4. Support online/offline presence
5. Store users' chats

NFRs:
1. Highly available chat service (99.99%)
2. Low latency message delivery when both online (p99 < 200ms)
3. Message durability required
4. Eventual consistency acceptable
5. 500M DAU, ~10 msg/day sent + 10 read, 10^5 sec/day → 50k/s avg writes & reads; peak 10x → 500k/s reads.

**Interviewer:** Math partially right but missed group fanout. 1 group message = 1 write but 500 deliveries. With 80% 1:1 and 20% group (avg 20 members) → 4.8× amplification. At 50k writes/s avg, deliveries ~240k/s avg, ~2.4M/s peak. Read receipts amplify again. Now give me: storage/year, concurrent online users.

**Aayush:** 10M concurrent users.

**Interviewer:** Real number probably higher (50-100M closer to WhatsApp). Show storage math.

**Aayush:** 500GB/day → 210TB/year.

**Interviewer:** ~158 TB/year actually (50k × 100B × 86400 = 432GB/day). Close enough. Text-only — media is 10-100x more, push to blob storage. Now HLD please. Lead with components and data flow.

**Aayush:** [Asked to update drawio with FRs and NFRs.]

**Interviewer:** [Updated.] HLD please.

**Aayush:** I plan on stating core entities and API design before HLD.

**Interviewer:** No. We drilled this 30 minutes ago — your #1 cost driver. HLD first; data model and APIs fall out naturally after. Walk me through components, data flow, online-to-online, online-to-offline.

**Aayush:** Client connects to chat service via WebSocket. Client picks recipient, sends message body. Chat service writes to DB. When recipient connects, message relayed. Presence inferred from active WebSocket connection.

**Interviewer:** Cartoon version. With 10M concurrent connections you need a fleet of chat nodes. Critical gaps: (A) Connection registry — how do you find which node holds Bob's socket? (B) Cross-node delivery — sender's node has the message; how does it reach recipient's node? (C) Group fanout — where does it happen? (D) Offline delivery — what query? Pick A and B first.

**Aayush:** [Asked to update drawio with version described so far.]

**Interviewer:** [Updated diagram with cartoon HLD: sender → Chat Service → DB → relay to recipient.] A and B please.

**Aayush:** A: Connection registry in Redis — entry added on WS connect, removed on disconnect. B: Redis pub/sub — when user connects, their node subscribes to their channel. Sender's node publishes to user's channel, subscriber relays.

**Interviewer:** Probes: (1) Pub/sub is fire-and-forget; reconnect/network blip = message lost. How do you make at-least-once? Hint: DB persistence fits in how? (2) Group fanout — publish to 500 user channels or one group channel? At 100k group msgs/s × 500 = 50M ops/s, Redis cluster dies. (3) Stale registry entries — node dies without cleanup. (4) Redis scale — single instance maxes ~100k ops/s, need cluster. Pick 1 and 2.

**Aayush:**
3. TTL on Redis entry + heartbeat to keep alive.
2. Publish to group channel; chat nodes subscribe on behalf of connected users.
1. Use Kafka for durability and replayability. Group and user partitions. Chat nodes subscribe to relevant partitions.

**Interviewer:** Solid progression. Probes:
- TTL/heartbeat: pick numbers; trade-off short vs long.
- Group channel: lifecycle — when subscribe, when unsubscribe?
- Kafka: how many partitions, partition key, consumer group structure, does Kafka replace pub/sub or layer?

**Aayush:** [Asked to update drawio with current design.]

**Interviewer:** [Updated: 2 chat nodes, Redis registry with TTL+heartbeat, Redis pub/sub channels, Kafka with partitions, Message DB.] Walk me end-to-end for (1) 1:1 message, (2) group message. And: does Kafka replace pub/sub?

**Aayush:** Alice connects to Node A, sends message to Bob. Message arrives at Node A, pushed to Kafka partition for Bob in user messages topic. Bob connected to Node B, Node B subscribes to Bob's Kafka partition, delivers via WebSocket. Kafka replaces Redis pub/sub.

**Interviewer:** Defend partitioning. Can't have a partition per user (500M users vs ~200k partition limit per cluster). Standard: hash userId to partition, many users share. Node consuming Bob's partition gets messages for ~500k users — must filter. How do you size N? For groups: publish to groupId partition or fan out 500 times? Offline catch-up — Kafka replay or DB query? Latency note: Kafka is 5-50ms vs sub-ms pub/sub — within your 200ms SLA but tight.

**Aayush:** Publish to groupId partition; recipient chat nodes pull from group partition.

**Interviewer:** Cleaner choice. Kafka consumer group issue: within a consumer group, each partition has one consumer. If nodes B, C, D are in the same group, only one gets each message. For fanout, options: (a) each node = own consumer group, (b) separate delivery service. Trade-offs? Plus: offline delivery, hot group problem.

**Aayush:** Each chat service node is its own consumer group — simplifies architecture.

**Interviewer:** Standard Kafka fan-out pattern. Trade-off: Kafka read throughput scales with N nodes. Acceptable. Still pending: (1) offline delivery — Kafka replay or DB query? (2) Hot group concern?

**Aayush:**
1. Kafka replay from offset.
2. Huge messages to deliver to a single node's connected users.

**Interviewer:** Kafka retention ~3-7 days; for weeks-offline users, fall back to Message DB. So both: Kafka for recent, DB for long-tail. Hot group: real fixes are shard a hot group across sub-partitions (lose ordering), batch delivery, dedicated hot-tier infra. You spotted the problem but didn't propose a fix. Share current time.

**Aayush:** 16:10.

**Interviewer:** [Delivered feedback.]

---

## Design Summary

**Requirements Gathered:**
- FRs: 1:1 and group (≤500) messages, reliable delivery online/offline, read receipts, presence, chat history.
- NFRs: 99.99%, p99 <200ms when both online, durable, eventual consistency, 500M DAU.
- Scale: 50k/s writes avg, 500k/s peak; 240k/s deliveries avg, 2.4M/s peak (4.8× fanout); 10M concurrent; ~200 TB/year text.

**High-Level Architecture:**
- Clients connect to Chat Node fleet via WebSocket.
- Redis Connection Registry: userId → nodeId, TTL + heartbeat.
- Kafka: durability + replay; partitions by userId and groupId; each Chat Node = own consumer group (fan-out).
- Message DB: durable history, offline delivery fallback for long-offline users.
- Redis Pub/Sub dropped in favor of Kafka-only.

**Key Design Decisions & Trade-offs:**
- Kafka replaces Redis pub/sub: accept 5-50ms latency vs sub-ms, for durability + replay.
- Publish group messages to groupId partition (not 500 user-channel publishes).
- Each chat node = own Kafka consumer group: simpler, accept N× read throughput.
- TTL + heartbeat on registry entries for liveness.
- Offline catch-up: Kafka for recent (3-7 day window), DB for long-tail.

**Scalability & Fault Tolerance Points:**
- Chat node fleet for 10M concurrent connections.
- Kafka partition fanout for groups.
- Registry TTL handles dead-node cleanup.

**Gaps / Missed Areas:**
- Did not cover API design.
- Did not cover data model / entities.
- Read receipts mechanism untreated.
- Presence broadcast (notifying contacts when status changes) not discussed.
- DB sharding for 200 TB/year not discussed.
- Hot group problem spotted but no solution proposed.
- Message ordering across nodes (clock skew, causal ordering) not addressed.
- Mobile push (APNs/FCM) for truly offline users not mentioned.
- Multi-region / replication not discussed.
- Diagram lagged — user drew nothing himself.

---

## Feedback Given

**Time Taken: 71 minutes** (target 45-60; 23 min improvement over prior round)

### Scorecard
| Area | Score | vs Last Round |
|---|---|---|
| Requirements | 7.5/10 | ↑ from 7 |
| HLD | 6.5/10 | ↑ from 6 |
| API Design | N/A (skipped) | — |
| Trade-offs | 6.5/10 | ↑ from 6 |
| Scalability | 6/10 | flat |
| Deep Dives | 6/10 | ↑ from 5.5 |
| Communication | 6/10 | ↑ from 5 |
| Diagram | 6.5/10 | ↑ from 6 |

### What Went Well
- NFRs with numbers volunteered upfront (drilled muscle showing up).
- BoE math correct first try; storage math solid.
- Accepted HLD-first correction without ego.
- Time discipline improved 25%.
- Arrived at canonical chat architecture (Redis registry + Kafka + DB + per-node consumer groups).

### What Still Cost
1. Still tried to skip HLD — opened with "let me do entities and APIs first."
2. Initial HLD was cartoon-level — needed multi-step probing to develop routing/registry/cross-node.
3. Didn't volunteer break/fix on own design (Redis pub/sub durability, Kafka consumer-group semantics, hot partitions — all surfaced by interviewer).
4. Hot group spotted but no solution proposed.
5. API design and data model entirely skipped due to time.
6. Diagram lagged verbal design throughout; user drew nothing.

### Areas Not Covered
- Read receipts mechanism.
- Presence broadcast.
- DB sharding strategy.
- Message ordering across nodes.
- Mobile push for truly offline users.
- Multi-region.

### Bottom Line
Measurable improvement vs prior round on numbers volunteered, BoE accuracy, time discipline, and acceptance of HLD-first correction. Same chronic patterns: data-model-first instinct, cartoon-then-pull-the-rest-out-of-you, no proactive break/fix, no self-drawing. Trajectory is right.
