# System Design Weaknesses
Last updated: 2026-09-29 (session 50 — system design round: Logging & Telemetry Pipeline)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Arithmetic slips in BoE / SLA math | 27 | 2026-09-29 |
| Asserts a traffic model without sanity-checking it | 15 | 2026-09-29 |
| Stops one step short of the number that decides it | 4 | 2026-09-29 |
| Produces zero NFRs / no numbers at all | 1 | 2026-08-22 |
| Never asks for scale givens when invited | 1 | 2026-08-22 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Response omits load-bearing fields (cursor, id, status) | 33 | 2026-08-19 |
| In-scope FR ships with no endpoint or mechanism | 10 | 2026-09-29 |
| No error/status semantics on any endpoint | 9 | 2026-09-29 |
| No idempotency key on a retryable write | 3 | 2026-08-20 |
| No position on cursor stability under concurrent writes | 1 | 2026-08-20 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 23 | 2026-09-29 |
| Hand-waves core algorithm (geo, sharding, consensus) | 13 | 2026-09-29 |
| Declines to attempt sizing before being pushed | 7 | 2026-09-29 |
| Moves on from a break he just identified | 2 | 2026-09-29 |
| Names a mitigation instead of resolving the break | 2 | 2026-09-29 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 26 | 2026-09-29 |
| Missing resilience patterns (DLQ, breaker, failover) | 16 | 2026-09-29 |
| Names a datastore category, never a concrete system | 3 | 2026-08-19 |
| Trade-offs on own inventions, none on dependencies | 1 | 2026-08-20 |
| Makes a scoping decision silently instead of stating it | 1 | 2026-08-20 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Over-runs requirements phase, starves the deep dive | 36 | 2026-09-29 |
| Doesn't volunteer break/fix in deep dives | 19 | 2026-09-29 |
| FRs restate the prompt instead of making choices | 2 | 2026-09-29 |
| Never states what is out of scope | 2 | 2026-09-29 |
| Abandons the round mid-phase instead of pushing through | 1 | 2026-08-22 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Mixed — self-raised batch idempotency key, cursor-with-reason, ack-after-enqueue durability and the createdAt hot shard; but HLD shipped with no query path and alert workers never got a data source | 2026-09-29 |
| Leads with trade-offs vs alternatives | Weak — Cassandra vs SQL justified by "only createdAt ordering needed" (contradicts own content-search FR); ClickHouse vs Elasticsearch asked twice, never answered | 2026-09-29 |
| Pushes scale until it breaks | Weak — three uncaught arithmetic errors (reads 10^4× high and "read heavy"; storage used peak rate; alert load 60× high); ES sizing stopped at "many shards"; "don't know" at 10× | 2026-09-29 |
| API as a designed contract | Mixed — batch ingest + Idempotency-Key + nextCursor with reason; alert rule has no evaluation window, PATCH has no path, no 429/backpressure semantics | 2026-09-29 |
| Operability / second-order concerns | Weak — consumer lag only when prompted at wrap-up; no loss detection, noisy-host partition, agent buffering, DLQ, alert dedupe, or cost | 2026-09-29 |
| Pace (core by mid, deep dive after) | Weak — HLD complete at 42 min, round 64 min; 11 follow-up exchanges ≈ half the round | 2026-09-29 |
