# System Design Round Transcript
**Date:** 2026-06-02
**Start Time:** 9:06
**End Time:** 10:33
**Duration:** 87 minutes
**Problem:** Design a Distributed Leaderboard

---

## Conversation Log

**Interviewer:** Welcome. Note your start time. Problem: Design a Distributed Leaderboard — a global ranking system for a large online game. Players accumulate scores; the system must show global Top-N, a player's own rank, and the neighbors around them in near real-time. Start with requirements gathering.

**Aayush:** Start time -> 9:06.

**Interviewer:** Recorded. Begin requirements gathering — FRs/NFRs and clarifying questions.

**Aayush:** So the users can see their rank, a global topN players and the local leaderboard?

**Interviewer:** Yes — those are the core FRs. Pushed: how do scores get in (write path)? Single global board or many (per-game/region/time-window)? Anything out of scope?

**Aayush:** 1. Scores get updated through events like games in the platform. 2. Do we need to support topN for a particular time window or all time? 3. Score change history for a player is out of scope.

**Interviewer:** Answered: support both all-time and daily + weekly. Asked for concrete NFRs with numbers.

**Aayush:** What scale can I consider — is 10M DAU okay?

**Interviewer:** 10M DAU is fine. Derive the rest yourself: writes/s, reads/s, storage — show assumptions and arithmetic.

**Aayush:** NFRs -> (1) HA 99.9% (~8.6 hrs/yr); (2) p99 < 200ms; (3) eventual consistency ~1min OK; (4) 10M DAU, 10 games/day/player ≈ 1k writes/s; each user loads leaderboard 5x/day, 20 rows/page -> 10k reads/s; read-heavy; peak 10x -> 100k reads/s.

**Interviewer:** Pressure-tested: writes ~1.16k/s OK. Reads = 50M/day ÷ 86400 ≈ 580/s avg — where did the 17x come from? A 20-row page is 1 request, not 20. Also justify the 10x peak, and does ~1min staleness apply equally to own-rank vs global Top-N?

**Aayush:** Yes ~5k reads/s. Peak is bursty after event completions. Consistency: the user sees their latest score but the global leaderboard takes time to reflect it.

**Interviewer:** Corrected — page = 1 read; ~5k/s is peak, ~580/s avg. Praised the own-rank-fresh vs global-eventual nuance. Locked NFRs. Moved to entities + HLD.

**Aayush:** (asked to add FRs to diagram) Core entities: Player, Leaderboard, PlayerScore — attributes deferred.

**Interviewer:** Added FRs/NFRs/entity+API placeholders to diagram. Probed: is PlayerScore per-game or cumulative? Does one row serve all three windows?

**Aayush:** PlayerScore is timestamped — one record per player per day, updated through the day; new record at day boundary. Daily uses that day's records; weekly uses past 7 days' records.

**Interviewer:** Probed all-time handling, and the core problem: ORDER BY across 10M players per read (and summing 7 rows for weekly) blows the latency budget. Asked for full HLD write + read path.

**Aayush:** Write: game-completion events -> Kafka (partition by playerId, durable) -> consumer updates PlayerScore rows. Read own score: GET /playerScores/:playerId?timeWindow={all-time|day|week} via API gateway (auth) -> leaderboard service queries SQL (index on playerId+createdAt), sums relevant rows. Top-N: service queries all rows in window, sums per player, sorts, returns Top-N — would fail at 10M scale, needs precomputation (deep dive).

**Interviewer:** Praised self-identified bottleneck. Probed: (1) endpoint returns score not rank — does it satisfy the FR? (2) ZINCRBY/at-least-once idempotency; (3) why partition by playerId?

**Aayush:** 1. Rank needs the leaderboard computed -> precompute it and update on score change. 2. Idempotency key = gameId + playerId in DB. 3. Partition by playerId ensures all a player's events land in one partition, ordering preserved.

**Interviewer:** All three correct. (Updated diagram with entities, API, and read/write architecture on request; later spaced sections out on request.) Deep dive: what data structure + store serves Top-N, rank, and neighbors? Complexity?

**Aayush:** Redis sorted set, key = date, value = sorted set of (score, playerId). 8 keys: 1 all-time + 7 daily; daily TTL 7 days. On event, consumer updates the date key's ZSET in O(log n) and adds to all-time. Rank for a day or all-time = O(log n). Weekly = merge 7 daily sets then get rank.

**Interviewer:** ZSET correct, complexities right. Broke it: weekly ZUNIONSTORE per read is O(N) — recreates the scan problem at 5k/s. How to avoid per-read union? Memory sizing? Neighbors command?

**Aayush:** (asked for hints on 1 and 3)

**Interviewer:** Hints — #1: trade read for write, use your ~1min staleness, do union off the read path or maintain incrementally. #3: once ZREVRANK gives rank r, get adjacent positions by rank-index.

**Aayush:** 1. Trade read latency for write — maintain weekly keys too; each write updates 1 daily, 1 weekly, 1 all-time key. Weekly TTL 7 days, daily 1 day.

**Interviewer:** Right trade. But incremental weekly: how does an old day leave the window? TTL on the whole key doesn't peel days. Rolling-7-days or calendar-week?

**Aayush:** Current calendar week, to keep it simple.

**Interviewer:** Sound — calendar-week keys reset cleanly; product implication: weekly rank resets at week start. Asked again for #3.

**Aayush:** ZREVRANGE with range +-k.

**Interviewer:** Correct. Bottlenecks/fault tolerance: (1) ZINCRBY isn't idempotent under replay — double counts; fix? (2) Redis is in-memory + source of truth — durability/HA + rebuild? (3) memory sizing?

**Aayush:** 1. Make the Redis update the last thing done. 2. Redis cluster for HA, circuit breaker on Redis, enable disk flushes when Redis goes down for recovery. 3. 1 GB RAM, one node is enough.

**Interviewer:** #1 — ordering doesn't fix replay (whole handler reruns; ZINCRBY fires again). Use the authoritative cumulative total in SQL + absolute ZADD instead of delta ZINCRBY. #2 — good instincts (circuit breaker, persistence) but persistence is configured up front; how to rebuild ZSETs from durable store on total loss? #3 — agreed; replicas add read-scaling + failover.

**Aayush:** 1. Yes — PlayerScore holds the cumulative score for the day, use that to set Redis (and store by day/week/all-time). 2. Query the DB to reconstruct the leaderboard, though expensive.

**Interviewer:** Both correct — absolute ZADD is idempotent under replay; DB rebuild is acceptable because total Redis loss is rare (batch aggregate PlayerScore rows; optional periodic snapshot). Wrapped up, asked for end time.

**Aayush:** 10:33.

**Interviewer:** Delivered feedback.

---

## Design Summary

**Requirements Gathered:**
- FRs: score updates via game-completion events; get own rank; get global Top-N (all-time/daily/weekly); get local leaderboard (K neighbors); out of scope: score-change history.
- NFRs: 99.9% availability; p99 < 200ms reads; own-rank fresh (read-after-write), global rankings eventual (~1 min); ~1.2k writes/s, ~580 reads/s avg, ~5k reads/s peak (bursty), read-heavy.

**High-Level Architecture:**
- Write: Game completion -> Kafka (partition by playerId, durable, ordered) -> consumer -> SQL PlayerScore (one row per player/day, cumulative; idempotency key gameId+playerId) + Redis ZSETs.
- Read: Client -> API Gateway (auth) -> Leaderboard Service -> Redis ZSETs (rank/Top-N/neighbors), SQL as durable store.
- Precompute: Redis Sorted Sets — daily, weekly (calendar week), all-time keys. ZADD/update O(log n), ZREVRANK O(log n), ZREVRANGE 0 N for Top-N, ZREVRANGE r-k r+k for neighbors.

**Key Design Decisions & Trade-offs:**
- Precompute ranks in Redis ZSETs to avoid O(N) scan/sort per read.
- Trade read cost for write cost: maintain weekly key incrementally instead of unioning 7 sets per read; justified by ~1min staleness NFR.
- Calendar-week keys for clean window rollover (weekly rank resets at week start).
- Kafka partition by playerId for ordering + single-owner-per-player.
- Idempotency: SQL via gameId+playerId; Redis via absolute ZADD of cumulative total (not delta ZINCRBY).

**Scalability & Fault Tolerance Points:**
- ~1GB all-time ZSET fits one node; single Redis primary handles 5k reads/s; replicas for read-scaling + failover.
- Circuit breaker on Redis; AOF/RDB persistence; rebuild ZSETs from durable SQL PlayerScore on total loss (rare, batch).

**Gaps / Missed Areas:**
- Reads BoE wrong first pass (rows vs requests).
- API response shapes never specified; no Top-N endpoint path; no pagination on Top-N.
- Endpoint returned score not rank until prompted.
- ZINCRBY idempotency bug not self-spotted; first fix ("Redis last") didn't address it.
- Precompute/Redis layer never added to the diagram.
- Asked for hints on #1 and #3 before attempting.
- Time overrun: 87 min.

---

## Feedback Given

# Feedback — Distributed Leaderboard

## Requirements Clarification — Strong (8/10)
Clean FR scoping via clarifying questions; concrete, well-reasoned NFRs; excellent own-rank-fresh vs global-eventual consistency nuance, later used to justify the weekly trade-off. Watch: reads BoE wrong first pass (rows vs requests); state avg vs peak unprompted.

## High-Level Architecture — Strong (8/10)
Clear write/read paths, concrete stores, justified Kafka + partition-by-playerId.

## API Design — Adequate (6/10)
Concrete endpoint with query param, but no response shapes, no Top-N path, no pagination; endpoint returned score not rank until prompted.

## Component Design & Trade-offs — Strong (8/10)
Redis ZSET correctly applied with right complexities; standout read-for-write weekly trade tied to the staleness NFR; deliberate calendar-week simplification.

## Scalability & Fault Tolerance — Good (7/10)
Right patterns (circuit breaker, replication, persistence, rebuild-from-SQL); correct memory BoE. Watch: persistence is up-front config; rebuild answer thin until prompted.

## Deep Dive Quality — Good (7/10)
Naive->break->fix arc present; self-volunteered the first break. But asked for hints on #1/#3 before attempting (neighbors were on the board); missed the ZINCRBY idempotency bug and first fix didn't address it.

## Communication & Process — Mixed (5/10)
Clear and structured, good deferral instincts. Headline issue: 87 minutes for a 45-60 min round — longest yet, partly diagram-tending mid-flow.

## Diagram Quality — Good
Faithful to verbal design (FRs, NFRs, entities, API, two-row architecture with directional labelled arrows). Gaps: the precompute/Redis ZSET layer never made it into the diagram; significant clock time spent on diagram formatting.

### Top 3 things to fix
1. Speed — get HLD + core deep dive out in ~35 min; diagram is a sketch, not a document.
2. Attempt before asking for hints, especially when the tool is already on the board.
3. Idempotency under replay should be reflexive (ZINCRBY delta vs ZADD absolute).
