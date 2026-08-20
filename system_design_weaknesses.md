# System Design Weaknesses
Last updated: 2026-08-20 (session 48 — system design round: Post Search)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asserts a traffic model without sanity-checking it | 14 | 2026-08-20 |
| Arithmetic slips in BoE / SLA math | 26 | 2026-08-20 |
| Mislabels an average as the peak | 1 | 2026-08-20 |
| Omits the freshness/staleness SLO on an async path | 1 | 2026-08-20 |
| Stops one step short of the number that decides it | 3 | 2026-08-19 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Response omits load-bearing fields (cursor, id, status) | 33 | 2026-08-19 |
| No idempotency key on a retryable write | 3 | 2026-08-20 |
| No position on cursor stability under concurrent writes | 1 | 2026-08-20 |
| In-scope FR ships with no endpoint or mechanism | 9 | 2026-08-17 |
| No error/status semantics on any endpoint | 8 | 2026-08-19 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 22 | 2026-08-13 |
| Moves on from a break he just identified | 1 | 2026-08-20 |
| Hand-waves core algorithm (geo, sharding, consensus) | 12 | 2026-08-13 |
| Names a mitigation instead of resolving the break | 1 | 2026-08-19 |
| Declines to attempt sizing before being pushed | 6 | 2026-08-19 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 25 | 2026-08-20 |
| Missing resilience patterns (DLQ, breaker, failover) | 15 | 2026-08-20 |
| Trade-offs on own inventions, none on dependencies | 1 | 2026-08-20 |
| Makes a scoping decision silently instead of stating it | 1 | 2026-08-20 |
| Names a datastore category, never a concrete system | 3 | 2026-08-19 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Over-runs requirements phase, starves the deep dive | 34 | 2026-08-19 |
| One-word non-answers to direct questions | 1 | 2026-08-20 |
| Doesn't volunteer break/fix in deep dives | 18 | 2026-08-08 |
| Earlier phases overrun; the API phase pays the bill | 5 | 2026-08-17 |
| Asks interviewer to make his own design decisions | 1 | 2026-08-19 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Strong — derived the scatter-gather break unprompted with full arithmetic (index size, shard count, projection, peak QPS multiply, "that does not work"), then volunteered both costs of his own fix without being asked. Did not raise ranking, freshness, or idempotency | 2026-08-20 |
| Leads with trade-offs vs alternatives | Mixed — every deep-dive answer carried its own cost named in the same breath (tail latency as slowest-of-32, bloom false positives, filter memory sized at tens of GB). But Cassandra, Kafka, Elasticsearch and Redis all arrived bare, with no alternative named for any of them; the asymmetry is trade-offs on his own inventions, none on his dependencies | 2026-08-20 |
| Pushes scale until it breaks | Strong — two full break→fix cycles, both self-initiated: scatter-gather across 20k shards at 150k peak QPS → time bucketing with early termination → write hotspot on the newest bucket → hash subsharding plus cold-bucket compaction. Then fixed the rare-term walkback with per-segment term filters and sized the filter's memory footprint unprompted | 2026-08-20 |
| API as a designed contract | Strong — request shapes on the write, cursor pagination with nextCursor and pageSize, a full 400/401/429/500 error table volunteered for the first time, JWT identity stated once unprompted, all first-pass with zero follow-ups. Missing: an idempotency key on POST /posts despite his own 99.9% write-availability NFR, and no position on cursor stability at 10k posts/s | 2026-08-20 |
| Operability / second-order concerns | Weak — no DLQ, no failover, no behaviour on an ES write rejection, no consumer-lag position, no cost discussion despite "retained forever" at 110 TB/year, and the closing freshness/monitoring question went unanswered entirely. The index path is fully asynchronous and has no observability story of any kind | 2026-08-20 |
| Pace (core by mid, deep dive after) | Strong — 22 min total against a 45-min reference, deep dive from minute 15, nothing truncated; a complete reversal of the prior sitting (56 min, deep dive starting at minute 45). Caveat recorded: the three deep-dive answers arrived ~1 min apart, each several hundred words with the trade-off pre-named, a different rhythm from the rest of the round where NFRs took 10 min and the ordering field needed a probe — treat as provisional until reproduced unassisted on an unseen problem | 2026-08-20 |
