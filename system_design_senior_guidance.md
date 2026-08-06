# Reaching a Senior Strong-Hire in System Design

A checklist of what separates **mid-level execution** (correct design, but led through it) from a **senior strong-hire** (self-driven, trade-off-led, reaches the genuinely hard problem). Distilled from mock rounds — review before every session.

> **One-sentence target:** Self-drive the traps, justify every choice against an alternative, push the scale until something breaks, and reach the genuinely hard problem with time to spare — instead of producing a correct design slowly and being led through the sharp edges.

---

## 1. Own the narrative — don't get led
The interviewer should feel they're *watching* you design, not *extracting* it.
- Surface the traps yourself before being asked: idempotency under at-least-once, read-vs-write cost, the missing piece in an endpoint, the part that won't scale.
- **Attempt before asking for hints.** A wrong-but-reasoned first attempt scores higher than asking for a nudge. Reason aloud and self-correct.
- Example: *"Note that `ZINCRBY` isn't idempotent under at-least-once delivery, so I'll keep the cumulative total in the DB and `ZADD` the absolute value instead."*

## 2. Lead with trade-offs, not just choices
Name a component **and** justify it against alternatives in the same breath, including what you're giving up.
- *"ZSET over a SQL `ORDER BY` because rank-by-index is O(log n) vs a full sort; over an approximate sketch because we need exact rank. Cost I accept: single-node memory, and a sorted set doesn't shard cleanly."*
- Every major box gets a "...and here's what I'm trading away."

## 3. Push the scale a notch and confront what breaks
Don't stop at the comfortable number.
- After the easy sizing ("fits one node"), ask: *"What at 10–50× this?"* and drive into what genuinely breaks.
- That break is usually the **senior-differentiating conversation** (e.g. for leaderboards: you can't shard a sorted set and keep a cheap global rank → bucketed/hierarchical ranking, approximate rank via score-histogram + local exact rank, sharded ZSETs with roll-up).

## 4. Treat the API as a designed contract
- Concrete **response shapes** with explicit fields (`{ playerId, rank, score, percentile }`, neighbor list structure).
- **Pagination** on list/Top-N endpoints — cursor vs offset and *why* (cursor for live/changing data).
- Correct HTTP verbs, idempotency on write/event contracts, versioning, error semantics.
- Name every endpoint, including the read ones.

## 5. Operability & second-order concerns (often the L5 vs L4 line)
- **Hot partitions / hot keys** — viral entity skews one shard; detect + mitigate.
- **Consumer lag & backpressure** during peak — what happens to freshness, is it within SLA?
- **Tie-breaking / edge cases** (equal scores, clock skew, late events).
- **Monitoring/alerting** — how do you *know* the system is stale or wrong? Own this part.
- **Cost** of the fleet at target scale.

## 6. Pace is itself a senior signal
- ~10 min on requirements + HLD, then drive hard into the 1–2 things that actually matter.
- Finishing the core with time left to go deep on one hard area **is** the signal.
- Don't spend the budget evenly and never reach the hard part. Hard cutoff discipline: target 45–50 min.

---

## Quick pre-round self-check
- [ ] Did I state avg **and** peak numbers, with the arithmetic, unprompted?
- [ ] Did I give response shapes + pagination for every endpoint?
- [ ] Did I justify each major component against an alternative?
- [ ] Did I raise the idempotency / consistency / failure traps before being asked?
- [ ] Did I push scale until something broke, and design for the broken case?
- [ ] Did I cover hot keys, lag/backpressure, tie-breaks, monitoring, cost?
- [ ] Am I on track to finish core design by ~30 min, deep dive by ~50?

---
---

# Reference material

Everything below is for study **between** rounds. It is an index to reach for, not a script to recite.

## Leveling: breadth ↓, depth ↑, proactivity ↑

The single most useful calibration. Interviewers don't just raise a bar uniformly — the *shape* of what they want changes with level.

| | Breadth | Depth | Proactivity |
|---|---|---|---|
| **Mid** | Wide questioning; expect "what does an API gateway do?" | Limited. Textbook understanding is acceptable | Drives requirements/API/schema; interviewer steers the rest |
| **Senior** ← *your bar* | Assumed. Explaining fundamentals unprompted **costs** you | Real depth in ~**two** areas, grounded in hands-on experience, not general statements | Recognizes limitations in your *own* design; steers toward what matters |
| **Staff+** | Assumed complete | Depth across several domains; can teach the interviewer something | Leads nearly the whole interview; foresees issues; behaves as a peer |

Two consequences worth internalizing:

- **Breadth is no longer the game.** Listing eight components you *could* add is a mid-level move. Picking two and going three layers deep is the senior one. You don't need depth everywhere — pick the two areas the problem actually turns on and own them.
- **Depth means experience, not vocabulary.** "We'd use a queue for backpressure" is general. "On the telemetry pipeline we saw consumer lag spike at deploy time because partition rebalancing stalled consumption, so we…" is depth. You have this material — D365 Durable Task Framework workflows, the telemetry pipeline, Azure region expansion, Rippling billing across timezones. **Use it by name.** A concrete war story is the cheapest senior signal available to you and you routinely leave it on the table.

## Non-functional requirements — the checklist

Vague NFRs are worth nothing. Every one needs a number and a context. *"The system should be low latency"* gives zero signal; *"search returns in <500ms p99"* gives all of it.

Walk this list, take the three or four that actually bind, and skip the rest out loud:

1. **Consistency vs availability** — which side of CAP, and *for which part* of the system (different subsystems can differ)
2. **Read/write ratio and traffic shape** — steady, diurnal, or spiky? What's the peak multiple?
3. **Latency** — per operation, p50 vs p99
4. **Durability** — how much data loss is tolerable, at what point does it become unacceptable
5. **Fault tolerance** — what survives an AZ loss, what degrades gracefully
6. **Scale** — users, QPS, storage growth per year
7. **Security / access control** — who can read what, tenancy boundaries
8. **Compliance & environment** — retention/residency, and client constraints (mobile battery, bandwidth)

Say which ones you're *not* going to design for. Explicit scoping is a senior move.

## The eight patterns — tell → approach

Most interview problems are one or two of these wearing a costume. Naming the pattern early tells the interviewer you've seen the shape before, and it tells you where the deep dive is going to be.

| Pattern | The tell | Start here → escalate to |
|---|---|---|
| **Pushing realtime updates** | chat, notifications, live dashboards, "users see it instantly" | polling → SSE (server→client) → WebSockets (bidirectional); pub/sub to decouple; consistent-hash ring for stateful servers |
| **Managing long-running tasks** | video encode, report generation, bulk import — anything over a few seconds | validate → enqueue (Kafka/Redis) → return job ID immediately; worker pool; retries + DLQ; status endpoint |
| **Dealing with contention** | ticket/seat booking, auctions, inventory, "two users grab the last one" | single-DB pessimistic lock or OCC with version column → distributed lock only when forced; hold reservations with TTL |
| **Scaling reads** | 100:1 read/write, feeds, product pages | index → read replicas → cache-aside (Redis) → CDN; then invalidation strategy, hot keys, stampede protection |
| **Scaling writes** | high write QPS, event ingest, telemetry | shard on a key with no hot spots → vertical partitioning → queue to absorb bursts → load shedding |
| **Handling large blobs** | video, images, documents | presigned URLs, client uploads direct to object storage; CDN with signed URLs; keep metadata in DB and reconcile |
| **Multi-step processes** | order fulfillment, payments, onboarding — a workflow that can fail halfway | workflow engine (Temporal / Step Functions / **DTF — your Microsoft experience**) or event sourcing; idempotency keys; audit trail |
| **Proximity / geospatial** | "near me", ride matching, local search | geospatial index (PostGIS, Redis GEO, Elasticsearch); geohash/quadtree regions; precompute cells for hot areas |

Contention, scaling writes, and multi-step processes are the three you have real production stories for. Reach for them.

## Numbers to know

Round numbers, stated confidently, beat precise numbers derived slowly. Use these to *decide something* — if a calculation doesn't change the design, say so and move on.

| Thing | Number |
|---|---|
| Redis | ~1 ms, 100k+ ops/sec/node |
| SQL DB | ~50k TPS ceiling, sub-5ms cached reads, single-digit-ms indexed lookups |
| Shard a database when | sustained writes over ~10k TPS, or the working set stops fitting in memory |
| Add/expand cache when | hit rate under ~80%, or read latency over ~1 ms |
| Cache speedup vs DB | 20–50× |
| Consistent hashing | ~1/N of keys move when a node joins; naive modulo moves ~everything |
| Network floor | NY→London ~80 ms RTT; same-region ~1 ms |
| Object storage | cheap and infinite; the cost is per-request and egress, not storage |
| Kafka partition | ~10 MB/s sustained; scale by partition count |
| A day | ~86k seconds — 1M/day ≈ 12/sec, 1B/day ≈ 12k/sec |

**Depth is graded in three tiers.** Mention the concept when you place the box (shallow). Explain its trade-off and when it applies (medium — this is what most rounds want). Walk implementation details and failure modes only when probed (deep). Volunteering tier-three detail unprompted burns the clock you need for the deep dive.

## The most common ways candidates fail

Ranked by how often they actually sink a round:

1. **Insufficiently exploring the problem** — designing before understanding, then solving the wrong thing
2. **Layering complexity too early** — never arriving at a complete working design. A simple complete system you then harden beats an elaborate incomplete one
3. **Focusing on trivial aspects** — polishing the schema or the auth flow while the hard problem goes untouched
4. **General statements instead of specifics** — the mid/senior line
5. **Memorized answers** — reciting a known architecture without connecting it to *these* requirements
