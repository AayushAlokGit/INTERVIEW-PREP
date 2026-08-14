# System Design Weaknesses
Last updated: 2026-08-14 (session 44 - design sprint: Logging & Telemetry Pipeline)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Arithmetic slips in BoE / SLA math | 24 | 2026-08-14 |
| Asserts a traffic model without sanity-checking it | 10 | 2026-08-14 |
| Omits read:write ratio, storage growth, durability | 6 | 2026-08-14 |
| Refuses to quantify when asked directly for a number | 6 | 2026-08-13 |
| Latency NFR stated without a percentile | 2 | 2026-08-13 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Response omits load-bearing fields (cursor, id, status) | 30 | 2026-08-14 |
| Misses read/delete endpoints until prompted | 7 | 2026-08-14 |
| No error/status semantics on any endpoint | 5 | 2026-08-14 |
| No request shapes or named body fields on writes | 1 | 2026-08-14 |
| No idempotency on the most contended write | 3 | 2026-08-13 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 22 | 2026-08-13 |
| Hand-waves core algorithm (geo, sharding, consensus) | 12 | 2026-08-13 |
| Declines to attempt sizing before being pushed | 5 | 2026-08-09 |
| Ends the round before the deep dive is reached | 1 | 2026-08-13 |
| Doesn't size the dependency or cost he introduces | 4 | 2026-08-10 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 23 | 2026-08-13 |
| Missing resilience patterns (DLQ, breaker, failover) | 13 | 2026-08-06 |
| Names a datastore without its key/partition design | 1 | 2026-08-13 |
| Weakens own NFR when it conflicts, instead of resolving | 2 | 2026-08-09 |
| Names a datastore category, never a concrete system | 2 | 2026-08-08 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Over-runs requirements phase, starves the deep dive | 32 | 2026-08-14 |
| Doesn't volunteer break/fix in deep dives | 18 | 2026-08-08 |
| Answers ~70% of a question, rest costs a round-trip | 8 | 2026-08-13 |
| Earlier phases overrun; the API phase pays the bill | 4 | 2026-08-14 |
| Serial clarifying questions burn the box before writing | 2 | 2026-08-14 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Weak | 2026-08-13 |
| Leads with trade-offs vs alternatives | Weak | 2026-08-13 |
| Pushes scale until it breaks | Weak | 2026-08-13 |
| API as a designed contract | Mixed — volunteered idempotency on every write and cursor pagination on every list, unprompted; but shipped zero request bodies, zero response fields, no error semantics, no search param on the search endpoint, and no way to obtain the id the metric read requires | 2026-08-14 |
| Operability / second-order concerns | Not observed | 2026-08-13 |
| Pace (core by mid, deep dive after) | Weak — all three phases landed (up from zero last sitting) but none inside its target: 10:50 of a 17:00 box before first content, entities took 5:07 to produce 12 words, API landed 1:38 past the buzzer. First passes were single-message and clean, so the failure is throughput on the thinking phases, not packaging | 2026-08-14 |
