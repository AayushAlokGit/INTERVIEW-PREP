# System Design Weaknesses
Last updated: 2026-08-19 (session 47 — system design round: Price Tracking Service)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Arithmetic slips in BoE / SLA math | 25 | 2026-08-19 |
| Asserts a traffic model without sanity-checking it | 13 | 2026-08-19 |
| Omits read:write ratio, storage growth, durability | 8 | 2026-08-19 |
| Defers the traffic model and never returns to it | 2 | 2026-08-19 |
| Stops one step short of the number that decides it | 3 | 2026-08-19 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Response omits load-bearing fields (cursor, id, status) | 33 | 2026-08-19 |
| No error/status semantics on any endpoint | 8 | 2026-08-19 |
| In-scope FR ships with no endpoint or mechanism | 9 | 2026-08-17 |
| Resource not addressable — id missing from path | 1 | 2026-08-19 |
| No request shapes or named body fields on writes | 2 | 2026-08-17 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 22 | 2026-08-13 |
| Hand-waves core algorithm (geo, sharding, consensus) | 12 | 2026-08-13 |
| Names a mitigation instead of resolving the break | 1 | 2026-08-19 |
| Declines to attempt sizing before being pushed | 6 | 2026-08-19 |
| Doesn't size the dependency or cost he introduces | 4 | 2026-08-10 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 24 | 2026-08-19 |
| Missing resilience patterns (DLQ, breaker, failover) | 14 | 2026-08-19 |
| Names a datastore category, never a concrete system | 3 | 2026-08-19 |
| Notices redundant work, then dismisses it unquantified | 1 | 2026-08-19 |
| Names a datastore without its key/partition design | 1 | 2026-08-13 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Over-runs requirements phase, starves the deep dive | 34 | 2026-08-19 |
| Doesn't volunteer break/fix in deep dives | 18 | 2026-08-08 |
| Every phase over reference; deep dive starts past 45 | 1 | 2026-08-19 |
| Asks interviewer to make his own design decisions | 1 | 2026-08-19 |
| Earlier phases overrun; the API phase pays the bill | 5 | 2026-08-17 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Mixed — volunteered naive→break→fix on the ingest path (single cron → fleet; cron doing fetch+notify+persist → cron publishes to Kafka for durability) and raised consumer idempotency before being asked. But deferred the traffic model and never returned to it, and needed a probe for hot reads, payload size, retailer rate limiting, and the repeat-notification trap that nobody surfaced all round | 2026-08-19 |
| Leads with trade-offs vs alternatives | Weak — one fully-formed trade-off (in-memory cache speed vs consistency cost) and one acknowledged (redundant price points vs detection granularity). Everything else arrived bare: "a DB like SQL" for 550TB of append-only time-ordered data with no time-series store named, Kafka with no alternative, sharding with no key-choice comparison | 2026-08-19 |
| Pushes scale until it breaks | Weak — every number came out under direct instruction; two 10× slips in the core traffic model (20M chart loads/day → 20/s instead of 200/s, and the read:write ratio built on it); handed the actual break (11MB per chart response, 2.2 GB/s egress) he answered with cursor pagination rather than confronting it, and never reached downsampling | 2026-08-19 |
| API as a designed contract | Mixed — JWT identity stated up front, request shapes on both writes, cursor pagination with `nextCursor` delivered. But two of three endpoints needed a structural probe to become correct (no subscription id in the PUT path; no payload-size position on an unbounded 2-year read), no error semantics anywhere, no list/delete endpoints, and the PUT response still reads `targetPrice = null` on the endpoint whose job is setting targetPrice | 2026-08-19 |
| Operability / second-order concerns | Mixed — retailer rate limiting answered well with the operational cost named (own distributed limiter + configs), read replicas for skewed chart reads correct. But no monitoring of any kind (no staleness metric, no consumer lag alerting, no DLQ), no cost, and no answer for a cron instance dying while holding a static product assignment — which is exactly what breaks his own 99.99% detection-availability NFR | 2026-08-19 |
| Pace (core by mid, deep dive after) | Weak — regression from the last sitting. Every single phase over reference: requirements +5, entities +7, API +9, HLD +10, deep dive +16. Total 56 min vs 45. The deep dive began at minute 45, the exact minute a real round ends, so every number and every scale conversation in this round would have gone unreached. Root cause is a single decision: deferring the traffic model at minute 13 moved 16 minutes of cheap requirements-phase arithmetic into the most expensive part of the clock | 2026-08-19 |
