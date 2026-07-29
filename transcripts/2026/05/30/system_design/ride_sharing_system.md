# System Design Round Transcript
**Date:** 2026-05-30
**Start Time:** 17:17
**End Time:** 18:29
**Duration:** 72 minutes
**Problem:** Design a Ride-Sharing System (Uber-like)

---

## Conversation Log

**Interviewer:** Note the time. Design a ride-sharing system (Uber-like). Riders request rides, get matched with nearby drivers, see driver moving in real-time, pay at end. Drivers go online, get requests, accept/decline, complete trips. Draw.io path provided.

**Aayush:** 17:17

**Interviewer:** Lead requirements gathering.

**Aayush:** What scale?

**Interviewer:** 100M DAU. Derive: rides/sec, concurrent drivers, location update rate, storage, etc.

**Aayush:** FRs:
1. Users request rides src→dest
2. Drivers in local radius receive request, accept/decline
3. Real-time location tracking matched→destination
4. Payment to driver at ride end

Out of scope: ratings, payment system design, multiple ride types.

NFRs:
1. 99.99% availability for matching
2. p99 <3s matching latency
3. Location updates every 2s
4. Strong consistency on matching (no double-match)
5. 100M DAU × 1 ride/day / 10^5 s = 1k/s avg, 10k/s peak
6. 10k concurrent active trips, 10k location writes/s

**Interviewer:** Scale error — Little's Law again. concurrent_trips = rate × duration. 1k/s × 15min trip = 900k concurrent avg, ~9M peak. Off by 100×. Location writes: (drivers online + active trips) × 0.5/s ≈ 7M/s peak. Fix NFRs.

**Aayush:** Updated: 5M concurrent drivers, 2.5M location writes/s.

**Interviewer:** Better. Now HLD: driver location pipeline, ride request flow, in-trip tracking.

**Aayush:** Bidirectional comms needed → SSE. Drivers push location to Kafka, partitioned by driverId. Consumer group batches writes to Location DB. Rider hits stateless ride-matching service behind LB, sends coordinates. Service queries drivers within radius. Filtered drivers shown request. Locking ensures only one accepts. Once accepted, ride-matching service subscribes to matched driver's Kafka partition, streams location to rider via SSE.

**Interviewer:** Critical probes:
1. SSE is one-way — drivers need bidirectional, use WebSocket.
2. THE big gap — geo-query is hand-waved. Naive scan = 50B comparisons/s. Real options: geohash/S2/H3 buckets, PostGIS, Redis GEORADIUS, in-memory grid. Which?
3. Location DB target unspecified.
4. Lock — where, what key, recovery?
5. SSE connection is state; how does match notification reach rider?
6. Broadcast vs sequential offer?

Pick #2 first.

**Aayush:** PostgreSQL with PostGIS extension for querying, but won't handle 2.5M writes/s.

**Interviewer:** Correct identification of write bottleneck. What's the alternative? Hint: most rides are intra-city; durability not needed per update.

**Aayush:** Geohash mapping location → city; Redis stores city → driver coords. But still concerned about 2.5M writes/s on Redis.

**Interviewer:** Miscount — that's the global number. Per-city shards each take ~50-100k/s. A single Redis primary handles that. And durability is moot: each update superseded 2s later, Kafka is the durable log, Redis is a real-time index. Architecture: Kafka → per-city consumer → Redis GEOADD; queries via GEOSEARCH. Two follow-ups: (1) Redis GEOSEARCH internals (geohash + sorted set)? (2) Cross-city boundary case?

**Aayush:** next probe

**Interviewer:** Skipping noted (will mark as gaps). Next: locking. D2 and D4 tap Accept at the same instant. Where lock lives, key, what loser sees, phone-dies case?

**Aayush:** [Asked to update diagram.]

**Interviewer:** [Drew HLD: Rider, Driver, LB, Ride Matching Service, Redis Geo-Index (per-city), Kafka (partitioned by driverId), Batching Consumer, Location DB.] Back to locking.

**Aayush:** DB locking on ride record. Ensures no 2 drivers can accept. DB releases lock if process crashes after acquiring.

**Interviewer:** Right pattern (conditional UPDATE) but you conflated transactional lock (FOR UPDATE) with committed write. UPDATE rides SET driver_id=? WHERE id=? AND status='offered' RETURNING — only one wins, loser sees 0 rows. But once committed, ride is permanently driver_id=D2. Phone-dies-after-commit case: no DB lock helps. Recovery?

**Aayush:** Use locking in a transaction so locks released automatically when transaction initiated on driver accept.

**Interviewer:** Misunderstanding. SELECT FOR UPDATE only protects between BEGIN and COMMIT. Once committed, lock released; row is permanently assigned. For phone-dies-after-commit, need separate liveness mechanism: heartbeat or trip-start confirmation within N seconds; if missing, watcher flips ride back to 'offered' and re-offers. Share current time.

**Aayush:** 18:29.

**Interviewer:** [Feedback delivered.]

---

## Design Summary

**Requirements Gathered:**
- FRs: ride request, radius driver offers, real-time location tracking, payment to driver at end.
- Out of scope: ratings, payment processing, ride types.
- NFRs: 99.99%, p99 <3s match, 2s location ping, strong consistency on match, 100M DAU.
- Scale: 1k/s avg rides, 10k/s peak; 5M concurrent drivers; 2.5M location writes/s.
- Little's Law correction needed (concurrent trips, location writes off by 100×).

**High-Level Architecture:**
- Rider Client → Load Balancer → Ride Matching Service (stateless).
- Driver Client → Kafka (driver-location stream, partitioned by driverId).
- Kafka → Batching Consumer → updates Redis Geo-Index (per-city sharded); Location DB removed by user.
- Ride Matching Service queries Redis via GEOSEARCH for nearby drivers.
- On match, Ride Matching Service subscribes to matched driver's Kafka partition; streams location to rider via SSE.

**Key Design Decisions & Trade-offs:**
- Kafka for location stream (durability + replay).
- Per-city sharded Redis as geo-index (lose durability on Redis side; Kafka is source of truth).
- DB conditional update for match locking (in-transaction lock proposed).
- Communication: SSE chosen (incorrectly — one-way).

**Scalability & Fault Tolerance Points:**
- Per-city Redis shards distribute the 2.5M/s write load.
- Kafka partition-by-driverId for ordered per-driver updates.
- Batching consumer reduces downstream write amplification.

**Gaps / Missed Areas:**
- Geo-query hand-waved initially; never returned to internals (Redis GEOSEARCH algorithm).
- Cross-city boundary edge case skipped.
- SSE chosen for bidirectional comms (wrong — should be WebSocket for drivers).
- Locking recovery (phone-dies-after-commit) misunderstood; needed heartbeat mechanism prompting.
- API design and data model entirely skipped.
- Surge pricing, ETA estimation, driver-rider chat, multi-region not covered.
- Driver supply-demand balancing (heat-maps) not covered.

---

## Feedback Given

**Time Taken: 72 minutes** (target 45-60; same overrun pattern as chat round)

### Scorecard
| Area | Score | vs Last Round |
|---|---|---|
| Requirements | 7/10 | ↓ slightly from 7.5 |
| HLD | 7/10 | ↑ from 6.5 |
| API Design | N/A (skipped) | — |
| Trade-offs | 6.5/10 | flat |
| Scalability | 6.5/10 | ↑ from 6 |
| Deep Dives | 6/10 | flat |
| Communication | 6.5/10 | ↑ from 6 |
| Diagram | 6.5/10 | flat (edited it once) |

### What Went Well
- Attempted HLD in one pass without three prompts (biggest behavioral improvement).
- Volunteered out-of-scope list explicitly (mature interviewing).
- Edited diagram himself (first time this session-pair).
- Spotted PostGIS write-throughput problem without prompting.
- Recovered from Little's Law correction fast.
- Per-city sharding came out (once nudged).

### What Cost
1. Little's Law gap — third recurrence in two days. Concurrent trips off 100×.
2. Geo-query hand-waved — central algorithm of ride-sharing not addressed.
3. SSE chosen for bidirectional comms (wrong tool — WebSocket needed).
4. Locking semantics confused: conflated in-tx FOR UPDATE with committed conditional update; missed heartbeat for phone-dies recovery.
5. Skipped Redis internals + cross-city probe ("next probe") — reads as disinterest or unknowing.
6. API design and data model skipped again (third round).
7. Time discipline flat at 72 min, plateaued.
8. Doesn't volunteer break/fix (chronic).

### Areas Not Covered
- Data model, API design.
- Cross-city boundary geo-query.
- Surge pricing.
- ETA estimation (map service integration).
- Driver-rider chat during trip.
- Trip completion / payment trigger flow.
- Multi-region / DR.
- Fraud detection.
- Driver supply/demand balancing.

### Bottom Line
Forward progress on process (HLD-first instinct, diagram editing, out-of-scope volunteered). Flat on time and depth gaps (Little's Law again, SSE/WebSocket confusion, hand-waved core algorithm). Knowledge inventory has visible holes — protocol limits, consumer-group semantics, SSE direction, geo-indexing, stock-vs-flow. Each is 30 min of study; cumulatively costs 1-2 points per round.
