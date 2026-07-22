# System Design Weaknesses
Last updated: 2026-06-29 (session 32 - Object Storage / S3 - teaching session)

## NFRs
| Weakness | Sessions | Last Seen |
|---|---|---|
| Cannot derive scale numbers independently | 16 | 2026-06-22 |
| Arithmetic slips in BoE / SLA math | 13 | 2026-06-22 |
| NFRs stated too late without prompting | 12 | 2026-06-29 |
| Conflates flow rate with stock (Little's Law) | 3 | 2026-05-30 |
| Misses fanout amplification in read/write math | 2 | 2026-06-07 |

## API Design
| Weakness | Sessions | Last Seen |
|---|---|---|
| Vague response shape (no explicit fields) | 18 | 2026-06-22 |
| Missing pagination on list endpoints | 4 | 2026-06-22 |
| Omits idempotency on write endpoints | 5 | 2026-06-06 |
| HTTP verb mistakes (PUT vs PATCH/DELETE) | 5 | 2026-06-10 |
| API design entirely skipped | 4 | 2026-05-30 |

## Deep Dives
| Weakness | Sessions | Last Seen |
|---|---|---|
| Asks for hints / leans on interviewer | 16 | 2026-06-22 |
| Hand-waves core algorithm (geo, sharding, consensus) | 9 | 2026-06-11 |
| Durability ordering: sync-vs-async confusion | 1 | 2026-06-15 |
| Spots problem but no solution proposed | 2 | 2026-05-30 |
| Misunderstands queue / consumer-group semantics | 2 | 2026-05-30 |

## Architecture & Trade-offs
| Weakness | Sessions | Last Seen |
|---|---|---|
| States choices without naming the alternative | 15 | 2026-06-22 |
| Missing resilience patterns (DLQ, breaker, failover) | 10 | 2026-06-10 |
| Picks store contradicting access pattern (mutable sort key) | 3 | 2026-06-19 |
| Doesn't self-catch hot-key/partition in own design | 1 | 2026-06-22 |
| Doesn't name database concretely | 3 | 2026-06-05 |

## Communication & Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Requires multiple prompts to fully articulate ideas | 21 | 2026-06-29 |
| Over-runs time budget (60-70 min vs 45-50) | 19 | 2026-06-19 |
| Doesn't volunteer break/fix in deep dives | 14 | 2026-06-15 |
| Ends round early / asks AI to finish design | 2 | 2026-06-22 |
| Defers answering via diagram-update requests | 6 | 2026-06-22 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Owns the narrative (self-raises traps) | Mixed | 2026-06-29 |
| Leads with trade-offs vs alternatives | Mixed | 2026-06-29 |
| Pushes scale until it breaks | Mixed | 2026-06-22 |
| API as a designed contract | Weak | 2026-06-29 |
| Operability / second-order concerns | Mixed | 2026-06-22 |
| Pace (core by mid, deep dive after) | Strong | 2026-06-22 |
