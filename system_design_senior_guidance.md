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
